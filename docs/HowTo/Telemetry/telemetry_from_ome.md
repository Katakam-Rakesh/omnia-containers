# Configure OpenManage Enterprise Telemetry

## Overview

OpenManage Enterprise (OME) publishes to Kafka topics that already exist.
Omnia deploys a Vector-OME bridge that matches topics using the configured OME
identifier and routes metrics through `vmagent-vector` to VictoriaMetrics and
logs through `vlagent-vector` to VictoriaLogs. Omnia does not deploy OME or
create the OME topics.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Enable OME with the Kafka collection target. The Vector-OME bridge derives
  the required VictoriaMetrics and VictoriaLogs sinks from its enabled
  channels; another source is not required to select those sinks.
- Have an OME instance that can reach the native Kafka LoadBalancer endpoint.
- Install OpenSSL on the OIM if OME requires the exported client certificate in
  PKCS#12 format.

## Procedure

1. Configure the OME source and bridge in `telemetry_config.yml`:

    ```yaml
    telemetry_sources:
      ome:
        metrics_enabled: true
        logs_enabled: true
        collection_targets:
          - kafka

    telemetry_bridges:
      vector_ome:
        metrics_enabled: true
        logs_enabled: true
        ome_identifier: "ome"
    ```

    Each bridge channel requires the corresponding OME source channel. Change
    `ome_identifier` only when the OME Kafka topic prefix differs; the bridge
    matches `<identifier>.*` topics.

2. Deploy Telemetry, then export the Kafka connection details:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags deploy
    ./omnia.sh --run telemetry --tags external_kafka
    ```

3. Create the client certificate file shown by the utility:

    ```bash title="Run on: OIM"
    cd /opt/omnia/telemetry/output/project_default/external_kafka
    openssl pkcs12 -export -out user.pfx -inkey user.key -in user.crt
    ```

    Adjust the directory for another data root or project.

4. In OME, open `Configuration -> Remote Connectivity`, enable Kafka
   connectivity, choose SSL authentication, set the generated
   `kafka.bootstrap_server`, upload `ca.crt` as the server certificate, and
   upload `user.pfx` as the client certificate.

## Verification

On the Kubernetes VIP, confirm that the bridge is running:

```bash title="Run on: Kubernetes VIP"
kubectl get pods -n telemetry -l app=vector-ome
```

Confirm the requested OME channels are `deployed` under `sources.ome` and
`bridges.vector_ome: deployed` in `telemetry_status.yml`.

These values confirm that the bridge resources are deployed. Query the
matching OME topics and the enabled Victoria sink to prove end-to-end data
flow.

## Next steps

- Retain the exported Kafka CA and client files securely for OME maintenance.

## Troubleshooting

- **The bridge validation fails:** Ensure OME uses only the `kafka` collection
  target and enable each source channel required by the corresponding bridge
  channel.
- **The metrics or logs bridge lacks a sink:** Confirm the corresponding
  Vector-OME channel is enabled. The bridge selection supplies the required
  VictoriaMetrics or VictoriaLogs sink.
- **No OME topics are consumed:** Confirm OME is publishing to the generated
  native Kafka endpoint and that topic names match the configured identifier.
- **The export utility fails:** Verify Kafka pods are Running and Ready and both
  Kafka LoadBalancer services have external IPs.
