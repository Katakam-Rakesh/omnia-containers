# Configure VAST Telemetry

## Overview

Omnia integrates an existing VAST Prometheus endpoint with the shared
VictoriaMetrics `vmagent`. It creates a `vast-external` Kubernetes Service and
Endpoints object and, when required, a `vast-telemetry-credentials` Secret.
Omnia does not deploy or configure the VAST system.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Provide a VAST endpoint that the Kubernetes cluster can reach.
- Enable the VAST Prometheus metrics API and know its port and path.
- For basic authentication, provide `vast_username` and `vast_password` when
  prompted.
- For CA-signed TLS, place the PEM CA certificate on the OIM and record its
  path.

## Procedure

1. Enable VAST metrics and VictoriaMetrics in `telemetry_config.yml`:

    ```yaml
    telemetry_sources:
      vast:
        metrics_enabled: true
        logs_enabled: false
        collection_targets:
          - victoria_metrics

    vast_configuration:
      vast_endpoint: "172.18.44.171"
      vast_metrics_port: 443
      metrics_path: "/api/prometheusmetrics/all"
      scrape_interval: "30s"
      scrape_timeout: "15s"
      tls_mode: "self_signed"
      vast_ca_cert_path: ""
      auth_mode: "basic"
    ```

    `tls_mode` accepts `self_signed` or `ca_signed`; `auth_mode` accepts
    `basic` or `none`. When `ca_signed` is selected, set
    `vast_ca_cert_path` to the PEM file.

2. Run validation and deployment:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags validate
    ./omnia.sh --run telemetry --tags deploy
    ```

3. To collect VAST logs, keep metrics enabled, set `logs_enabled: true`, add
   `victoria_logs` to `collection_targets`, deploy Telemetry, and export the
   VLAgent target. The source role is imported only when metrics are enabled;
   a logs-only configuration is not supported.

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags external_victoria
    ```

    Configure the existing VAST system to send logs to the generated
    `vlagent.syslog_endpoint`. The Telemetry source exposes this endpoint but
    does not configure VAST itself.

    VAST log forwarding is an external system configuration step; the
    Telemetry source does not deploy a VAST log collector.

## Verification

On the Kubernetes VIP, confirm that the service and endpoints were created:

```bash title="Run on: Kubernetes VIP"
kubectl get service vast-external -n telemetry
kubectl get endpoints vast-external -n telemetry
```

Confirm `sources.vast.metrics: deployed` in `telemetry_status.yml`. This status
records the integration resource state; verify actual metric ingestion from
VictoriaMetrics separately for an end-to-end check. When logs are enabled,
`sources.vast.logs: deployed` records the configured log path; verify that the
VAST system is actually sending data.

## Next steps

- Use [Export VictoriaMetrics Connection Details](configure_external_victoria.md)
  to obtain the query endpoint and UI URL.

## Troubleshooting

- **The endpoint is rejected:** Set a non-empty VAST IP address and a port from
  `1` through `65535`.
- **Credentials are missing:** Supply VAST credentials when `auth_mode: basic`.
- **Credentials are requested with `auth_mode: none`:** The current credential
  collection is gated by enabled VAST metrics, not by `auth_mode`. Complete the
  prompt while this source behavior remains in place.
- **The CA file is rejected:** With `tls_mode: ca_signed`, provide an existing
  PEM certificate path on the OIM.
- **Deployment fails with `auth_mode: none` and self-signed TLS:** The current
  source can render an empty Secret while still attempting to apply it. Use
  basic authentication or CA-signed TLS until that source limitation is fixed.
- **No metrics arrive:** Confirm the VAST endpoint and metrics path are
  reachable from Kubernetes and that authentication and TLS settings are
  correct.
- **Logs do not arrive:** Confirm that VAST is sending to the exported VLAgent
  syslog endpoint and that `victoria_logs` remains in its collection targets.
