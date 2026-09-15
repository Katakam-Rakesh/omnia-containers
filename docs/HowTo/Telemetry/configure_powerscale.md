# Configure PowerScale Telemetry

Configure deployment of PowerScale Telemetry to collect storage performance
metrics and logs from Dell PowerScale storage nodes.

## Overview

PowerScale Telemetry collects storage performance metrics and logs. It includes
the following components:

### Components

- **CSM Metrics for PowerScale** -- Queries the OneFS API and emits metrics to
  an OpenTelemetry Collector.
- **OpenTelemetry Collector** -- Receives metrics from CSM Metrics and exposes
  a Prometheus endpoint for scraping.
- **vmagent** -- Scrapes the OpenTelemetry Collector Prometheus endpoint over
  plain HTTP and forwards metrics to VictoriaMetrics.
- **VLAgent** -- Receives plaintext PowerScale syslog events on TCP or UDP port
  514 and forwards them to VictoriaLogs.
- **CSI Driver for Dell PowerScale** -- Required for Omnia-orchestrated
  deployment mode.
- **cert-manager** -- Required for TLS certificate management in
  Omnia-orchestrated mode.

### Data flow

```text
PowerScale nodes → CSM Metrics PowerScale → OTEL Collector → vmagent (shared) → VictoriaMetrics
PowerScale nodes → syslog → VLAgent → VictoriaLogs
```

### Supported metrics and logs

Metrics:

| Category | Metrics collected |
|---|---|
| Performance | Protocol-level IOPS (NFS, SMB, S3), throughput (bytes/s), and read/write latency |
| Capacity | Total cluster capacity, used capacity, available capacity, and per-node capacity |
| Health | Node online/offline status, disk health, cluster rebalance status, and protection group status |
| Topology | Cluster node membership, node roles, interconnect layout, and protection domain mapping |

For the complete list of PowerScale Telemetry metrics, see the
[PowerScale Metrics reference](../../Reference/Metrics/powerscale_metrics.md)
and [Dell CSM Observability PowerScale metrics](https://dell.github.io/csm-docs/docs/concepts/observability/metrics/powerscale/).

Logs:

| Category | Logs collected |
|---|---|
| System Events | Capacity warnings, disk failures, node state changes, and protocol errors |
| Labels | Events labeled with host or cluster, severity, and facility |

### Health monitor metrics

When the CSI PowerScale health monitor is enabled with
`controller.healthMonitor.enabled: true` and `node.healthMonitor.enabled: true`
in the CSI PowerScale `values.yaml`, Omnia collects the following additional
health metrics.

PV metrics:

- `powerscale_volume_status` -- PV phase (`1=Bound`, `0=Other`), labeled by
  `pv_name` and `phase`.
- `powerscale_volume_count` -- Total PowerScale PVs by phase.
- `powerscale_volume_capacity_bytes` -- PV capacity in bytes.
- `powerscale_volume_info` -- PV metadata, including name, phase, storage
  class, reclaim policy, access modes, volume handle, PVC name, and namespace.
- `powerscale_volume_age_seconds` -- Seconds since PV creation.

PVC metrics:

- `powerscale_pvc_status_phase` -- PVC phase (`1=Bound`, `0=Other`), labeled by
  PVC name, namespace, and phase.
- `powerscale_pvc_requested_bytes` -- Requested PVC storage in bytes.
- `powerscale_pvc_count` -- Total PowerScale PVCs by phase.

Health event metrics:

- `powerscale_volume_health_abnormal` -- Volume condition (`1=abnormal`,
  `0=healthy`), labeled by PVC name, namespace, and PV name.
- `powerscale_volume_abnormal_events_total` -- Total
  `VolumeConditionAbnormal` events.
- `powerscale_node_failure_events_total` -- Total node failure events.

Node metrics:

- `powerscale_node_ready` -- Node Ready condition (`1=True`, `0=False`).

Storage class metrics:

- `powerscale_storageclass_info` -- StorageClass metadata, including the
  provisioner, reclaim policy, volume binding mode, and volume expansion flag.

Aggregate summary:

- `powerscale_total_capacity_bytes` -- Total capacity of all PowerScale PVs in
  bytes.

### Transport security and authentication

Security is configured independently for each hop:

- **PowerScale OneFS API to CSM Metrics:** CSM Metrics reads the endpoint and
  credentials from the copied `isilon-creds` Secret. OneFS server-certificate
  verification is controlled by
  `karaviMetricsPowerscale.isiClientOptions.isiSkipCertificateValidation` in
  the supplied CSM Observability values file. When Karavi Authorization is
  enabled, its proxy certificate and token Secrets are also copied from the
  `isilon` namespace.
- **OpenTelemetry Collector to vmagent:** The generated `VMServiceScrape`
  selects the collector's `prometheus` service port (8889) and `/metrics`
  path. It does not set `scheme: https`, a bearer token, `basicAuth`, or a TLS
  configuration, so this in-cluster scrape uses unauthenticated HTTP.
- **vmagent to VictoriaMetrics:** In cluster mode, the generated VMAgent remote
  write uses HTTPS and verifies the CA from the `victoria-tls-certs` Secret.
  This setting secures the sink hop; it does not make the collector scrape
  HTTPS.
- **PowerScale to VLAgent:** The exported target is the VLAgent LoadBalancer on
  plaintext syslog TCP or UDP port 514. The current manifest does not expose a
  TLS syslog listener on port 6514.
- **VLAgent to VictoriaLogs:** The generated VLAgent uses HTTPS with the
  Victoria CA when VictoriaLogs TLS is enabled, and HTTP otherwise.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Deploy the PowerScale CSI driver in the `isilon` namespace and ensure its pods
  are running. The `external-health-monitor-controller` is required for the CSI
  volume exporter health metrics.
- Provide a CSM Observability values YAML containing
  `karaviMetricsPowerscale.image`, `otelCollector.image`, and
  `cert-manager.enabled: true`. Do not enable PowerFlex, PowerStore, or PowerMax
  metrics in this file.
- Keep `images.powerscale.csm_metrics` and
  `images.powerscale.otel_collector` in `telemetry_packages.yml`; in offline
  mode, values-file image versions must match the package manifest.
- Provide `csi_username` and `csi_password` when prompted.

## Procedure

1. Configure the PowerScale source and its required metrics target in the
   `telemetry_config.yml` input file:

    ```yaml
    telemetry_sources:
      powerscale:
        metrics_enabled: true
        logs_enabled: false
        collection_targets:
          - victoria_metrics

    powerscale_configurations:
      otel_collector_storage_size: "5Gi"
      csm_observability_values_file_path: ""
    ```

    !!! note

        The default value of `csm_observability_values_file_path` is empty.
        When PowerScale metrics are enabled, replace it with the absolute path
        to an existing CSM Observability `values.yaml` file before running
        validation or deployment.

2. To prepare PowerScale log ingestion as well, keep metrics enabled, set
   `logs_enabled: true`, and add `victoria_logs` to `collection_targets`.
   The root workflow imports the PowerScale source only when
   `metrics_enabled: true`; a logs-only configuration is not supported.

3. Keep the `csm_metrics_powerscale_storage` and
   `csi_volume_exporter_storage` sections in `telemetry_storage_config.yml`.

4. Run the precheck and deployment:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags precheck
    ./omnia.sh --run telemetry --tags deploy
    ```

5. When logs are enabled, export the generated VLAgent target and PowerScale
   `isi audit` commands:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags external_victoria
    ```

    Run the commands recorded under `powerscale.isi_audit_commands` in the
    generated connection-details file on the PowerScale system.

## Verification

On the Kubernetes VIP, verify the same resources inspected by deployment:

```bash title="Run on: Kubernetes VIP"
kubectl get pods -n telemetry -l app.kubernetes.io/name=karavi-metrics-powerscale
kubectl get deployment otel-collector -n telemetry
```

Confirm `sources.powerscale.metrics: deployed` in `telemetry_status.yml`. When
logs are enabled, confirm `sources.powerscale.logs: deployed` and check that the
generated external Victoria file reports `vlagent.available: true`.

The log status confirms that the shared VLAgent is available; it does not
configure PowerScale log forwarding or prove ingestion. Run the exported
`isi audit` commands and query VictoriaLogs for an end-to-end check. Likewise,
query VictoriaMetrics to confirm that PowerScale metrics are being ingested.

## Next steps

- Use [Export Victoria Connection Details](configure_external_victoria.md) to
  obtain write, query, and syslog endpoints.

## Troubleshooting

- **Validation cannot find the values file:** Set an existing path in
  `csm_observability_values_file_path` and ensure it contains valid YAML.
- **The values file is rejected:** Enable cert-manager, provide the required
  PowerScale and OTEL images, disable unsupported storage metrics, and align
  image versions with `telemetry_packages.yml` in offline mode.
- **PowerScale precheck warns about privileges:** Verify the PowerScale account
  has the permissions required by the enabled metrics or log path.
- **CSI volume exporter is skipped:** Enable the external health monitor in the
  PowerScale CSI driver and rerun Telemetry.
