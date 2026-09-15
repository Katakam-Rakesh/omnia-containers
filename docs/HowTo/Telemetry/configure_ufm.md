# Configure UFM Telemetry

## Overview

Omnia integrates an existing NVIDIA UFM Prometheus endpoint with the shared
VictoriaMetrics `vmagent`. It creates a `ufm-external` Kubernetes Service and
Endpoints object and, when required, a `ufm-telemetry-credentials` Secret.
Omnia does not deploy or configure the UFM appliance.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Provide a UFM IP address that the Kubernetes cluster can reach.
- Enable the UFM Prometheus endpoint on the appliance and know its port.
- For basic authentication, provide `ufm_username` and `ufm_password` when
  prompted.
- For CA-signed TLS, place the PEM CA certificate on the OIM and record its
  path.

## Procedure

1. Enable UFM metrics and VictoriaMetrics in `telemetry_config.yml`:

    ```yaml
    telemetry_sources:
      ufm:
        metrics_enabled: true
        logs_enabled: false
        collection_targets:
          - victoria_metrics

    ufm_configuration:
      ufm_endpoint: "172.20.44.180"
      ufm_metrics_port: 9001
      scrape_interval: "30s"
      scrape_timeout: "15s"
      tls_mode: "self_signed"
      ufm_ca_cert_path: ""
      auth_mode: "basic"
    ```

    `tls_mode` accepts `self_signed` or `ca_signed`; `auth_mode` accepts
    `basic` or `none`. When `ca_signed` is selected, set
    `ufm_ca_cert_path` to the PEM file.

2. Run validation and deployment:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags validate
    ./omnia.sh --run telemetry --tags deploy
    ```

3. To collect UFM logs, keep metrics enabled, set `logs_enabled: true`, add
   `victoria_logs` to `collection_targets`, deploy Telemetry, and export the
   VLAgent target. The source role is imported only when metrics are enabled;
   a logs-only configuration is not supported.

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags external_victoria
    ```

    Configure the existing UFM appliance to send logs to the generated
    `vlagent.syslog_endpoint`. The Telemetry source exposes this endpoint but
    does not configure UFM itself.

    UFM log forwarding is an external appliance configuration step; the
    Telemetry source does not deploy a UFM log collector.

## Verification

On the Kubernetes VIP, confirm that the service and endpoints were created:

```bash title="Run on: Kubernetes VIP"
kubectl get service ufm-external -n telemetry
kubectl get endpoints ufm-external -n telemetry
```

Confirm `sources.ufm.metrics: deployed` in `telemetry_status.yml`. This status
records the integration resource state; verify actual metric ingestion from
VictoriaMetrics separately for an end-to-end check. When logs are enabled,
`sources.ufm.logs: deployed` confirms that VLAgent is running, not that the UFM
appliance has begun sending logs.

## Next steps

- Use [Export VictoriaMetrics Connection Details](configure_external_victoria.md)
  to obtain the query endpoint and UI URL.

## Troubleshooting

- **The endpoint is rejected:** Set a non-empty UFM IP address and a port from
  `1` through `65535`.
- **Credentials are missing:** Supply UFM credentials when `auth_mode: basic`.
- **Credentials are requested with `auth_mode: none`:** The current credential
  collection is gated by enabled UFM metrics, not by `auth_mode`. Complete the
  prompt while this source behavior remains in place.
- **The CA file is rejected:** With `tls_mode: ca_signed`, provide an existing
  PEM certificate path on the OIM.
- **Deployment fails with `auth_mode: none` and self-signed TLS:** The current
  source can render an empty Secret while still attempting to apply it. Use
  basic authentication or CA-signed TLS until that source limitation is fixed.
- **No metrics arrive:** Confirm the UFM endpoint is reachable from Kubernetes
  and that its authentication, TLS mode, scrape interval, and timeout are
  correct.
- **Logs do not arrive:** Confirm that UFM is sending to the exported VLAgent
  syslog endpoint and that `victoria_logs` remains in its collection targets.
