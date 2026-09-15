# Configure UFM Telemetry

Configure NVIDIA Unified Fabric Manager (UFM) to securely stream Telemetry
metrics and logs to the Service Kubernetes cluster.

## Overview

UFM Telemetry collects InfiniBand fabric metrics and logs from an existing
NVIDIA UFM appliance.

### Components

- **UFM Prometheus Exporter** -- Exposes InfiniBand metrics over a
  Prometheus-compatible HTTPS endpoint. The default port is `9001`.
- **vmagent (shared)** -- Scrapes the UFM exporter over TLS and forwards the
  metrics to VictoriaMetrics.
- **VMServiceScrape** -- Defines the UFM scrape target, authentication, TLS,
  interval, and timeout for vmagent.
- **VLAgent** -- Receives RFC 3164 or RFC 5424 syslog messages from UFM and
  sends them to VictoriaLogs.
- **Kubernetes Service and Endpoints** -- Represent the external UFM appliance
  as the `ufm-external` service in the `telemetry` namespace.

Omnia does not deploy or configure the UFM appliance.

### Data flow

```text
UFM Fabric Manager -> UFM Prometheus Exporter -> vmagent (shared) -> VictoriaMetrics
UFM Fabric Manager -> syslog -> VLAgent -> VictoriaLogs
```

### Supported metrics and logs

| Metrics category | Metrics collected |
|---|---|
| Port state | InfiniBand port operational state, including up, down, and disabled |
| Traffic counters | Transmit and receive rates, bytes per second, and packets per port |
| Error counters | Symbol errors, link recovery and link-down events, VL15 drops, and excessive buffer overruns |
| Fabric topology | Switch information, port mappings, node GUIDs, and LIDs |
| Telemetry health | Scrape success, scrape duration, and ingestion latency |

For the complete list, see the
[UFM Metrics reference](../../Reference/Metrics/ufm_metrics.md).

| Log category | Logs collected |
|---|---|
| Fabric events | Topology changes, port transitions, errors, and warnings |
| Manager events | Subnet Manager, SHARP, and UFM health events |
| Labels | Hostname, severity, and facility metadata |

UFM metrics and logs are controlled independently by `metrics_enabled` and
`logs_enabled`.

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

2. Run the Telemetry precheck. Choose one execution method; do not run both
   commands for the same operation.

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry --tags precheck
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source /opt/omnia/activate-omnia.sh
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml --tags precheck
        ```

3. Validate the Telemetry inputs and collect the required credentials:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry --tags validate
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source /opt/omnia/activate-omnia.sh
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml --tags validate
        ```

4. Deploy the enabled Telemetry configuration:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry --tags deploy
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source /opt/omnia/activate-omnia.sh
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml --tags deploy
        ```

5. To run validation and deployment in one invocation, omit the tag:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source /opt/omnia/activate-omnia.sh
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml
        ```

    The untagged flow does not run the opt-in precheck. Run step 2 separately
    when an environment precheck is required.

6. To collect UFM logs, keep metrics enabled, set `logs_enabled: true`, add
   `victoria_logs` to `collection_targets`, deploy Telemetry, and export the
   VLAgent target. The source role is imported only when metrics are enabled;
   a logs-only configuration is not supported.

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry --tags external_victoria
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source /opt/omnia/activate-omnia.sh
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml --tags external_victoria
        ```

    Configure the existing UFM appliance to send logs to the generated
    `vlagent.syslog_endpoint`. The Telemetry source exposes this endpoint but
    does not configure UFM itself.

    UFM log forwarding is an external appliance configuration step; the
    Telemetry source does not deploy a UFM log collector.

## Verification

### Verify UFM Telemetry pods

1. Verify that the VictoriaMetrics pods are running:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get pods -n telemetry -o wide | grep vm
    ```

    ![VictoriaMetrics pods](../../assets/images/verify_umf_telemetry_1.png)

2. Verify that the VictoriaMetrics services are running:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get service -n telemetry -o wide | grep vm
    ```

    ![VictoriaMetrics services](../../assets/images/verify_umf_telemetry_2.png)

3. Check the shared vmagent logs for successful UFM scrapes:

    ```bash title="Run on: Kubernetes control plane"
    VMAGENT_POD=$(kubectl get pods -n telemetry \
      -l app.kubernetes.io/name=vmagent \
      -o jsonpath='{.items[0].metadata.name}')
    kubectl logs "$VMAGENT_POD" -n telemetry -c vmagent --tail=50
    ```

    ![vmagent logs](../../assets/images/verify_umf_telemetry_3.png)

4. Confirm that the Kubernetes service and endpoints for the external UFM
   appliance were created:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get service ufm-external -n telemetry
    kubectl get endpoints ufm-external -n telemetry
    ```

### View UFM metrics in VictoriaMetrics UI

1. Identify the external `vmselect` service:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get svc -n telemetry | grep vmselect
    ```

    ![vmselect service](../../assets/images/verify_umf_telemetry_4.png)

2. Export the VictoriaMetrics connection details:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry --tags external_victoria
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source /opt/omnia/activate-omnia.sh
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml --tags external_victoria
        ```

3. Open the URL recorded in `victoria_metrics.endpoints.vmselect.ui_url` in
   the following file:

    ```text
    $OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_victoria/external_victoria_connect_details.yml
    ```

4. Query a UFM metric, such as `infiniband_CBW`, to confirm that UFM metrics
   are reaching VictoriaMetrics. To filter InfiniBand metrics by their source
   labels, use:

    ```promql
    {source="ufm", subsystem="infiniband"}
    ```

    ![UFM metrics in VMUI](../../assets/images/verify_umf_telemetry_5.png)

### View UFM logs in VictoriaLogs

Complete these steps only when UFM log collection is enabled.

1. Verify that VLAgent and VictoriaLogs services are running:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get svc -n telemetry | grep -E '(vlagent|victoria-logs)'
    ```

    ![VLAgent and VictoriaLogs services](../../assets/images/view_umf_telemetry_1.png)

2. Identify the external `vlselect` service:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get svc -n telemetry | grep vlselect
    ```

    ![vlselect service](../../assets/images/view_umf_telemetry_2.png)

3. If needed, export the Victoria connection details by using either command
   shown in step 2 of the previous section. Open the URL recorded in
   `victoria_logs.endpoints.vlselect.ui_url` in:

    ```text
    $OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_victoria/external_victoria_connect_details.yml
    ```

4. Query `ufm` to confirm that UFM logs are reaching VictoriaLogs.

    ![UFM logs in VictoriaLogs](../../assets/images/view_umf_telemetry_3.png)

Confirm `sources.ufm.metrics: deployed` in `telemetry_status.yml`. This status
records the integration resource state; successful metric queries provide the
end-to-end validation. When logs are enabled, `sources.ufm.logs: deployed`
confirms that VLAgent is running; a successful log query confirms that the UFM
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
