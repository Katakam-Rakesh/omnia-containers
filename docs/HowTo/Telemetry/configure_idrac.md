# Configure iDRAC Telemetry

## Overview

iDRAC Telemetry collects hardware metrics from Dell servers using the
integrated Dell Remote Access Controller (iDRAC). iDRAC Telemetry includes the
following components:

### Components

- **iDRAC Collector** -- Polls each server's Redfish endpoint for hardware
  metrics. Runs as a Kubernetes pod in the `telemetry` namespace.
- **ActiveMQ** -- Internal message broker used by the iDRAC collector to
  decouple metric collection from downstream routing.
- **KafkaPump** -- Routes iDRAC metrics from ActiveMQ to the Kafka `idrac`
  topic.
- **VictoriaPump** -- Routes iDRAC metrics from ActiveMQ to VictoriaMetrics
  through VMAgent.
- **VMAgent** -- Forwards metrics to the VictoriaMetrics cluster (`vminsert`).
- **MySQL Database** -- Stores iDRAC telemetry metadata. Storage size is
  configurable through
  [`telemetry_storage_config.yml`](../../Reference/Configuration/telemetry_storage_config.md).

### Data flow

```text
iDRAC (BMC) → iDRAC Collector → Kafka
iDRAC (BMC) → iDRAC Collector → VMAgent → VictoriaMetrics
```

### Supported metrics

| Category | Metrics collected |
|---|---|
| Thermal | Inlet temperature, exhaust temperature, CPU temperature, fan speeds |
| Power | System power consumption, PSU input/output power, power capping status |
| Storage Health | Physical disk status, virtual disk health, controller health, SMART data |
| CPU/Memory | Correctable/uncorrectable ECC errors, CPU utilization, DIMM health |
| System Events | Hardware alerts, lifecycle events, firmware status |

For the complete list of iDRAC telemetry metrics, see the
[Dell iDRAC Telemetry Reference Guide](https://dl.dell.com/content/manual43363890-dell-idrac-telemetry-reference-guide.pdf?language=en-us)
and [iDRAC Telemetry Reference Tools](https://github.com/dell/iDRAC-Telemetry-Reference-Tools).

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Provide the following input file with the header
  `BMC_IP,GROUP_NAME,PARENT`, and set its path in
  `idrac_telemetry_configurations.bmc_group_data_path`:

  ```text
  $OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/bmc_group_data.csv
  ```

- Ensure the BMCs are reachable from at least one service Kubernetes worker.
  If the worker cannot be reached over SSH, the source falls back to validating
  BMCs from the control-plane VIP.
- Have one common `bmc_username` and `bmc_password` for the BMCs, plus
  `mysqldb_user`, `mysqldb_password`, and `mysqldb_root_password`.

  These credentials are requested only when iDRAC metrics are enabled. They are
  stored in the encrypted project file
  `<OMNIA_DATA_PATH>/telemetry/input/<project>/telemetry_credentials.yml` and
  deployed to the `mysqldb-credentials` Kubernetes Secret.

## Procedure

1. In the project `telemetry_config.yml`, enable iDRAC and retain both required
   targets:

    ```yaml
    telemetry_sources:
      idrac:
        metrics_enabled: true
        collection_targets:
          - victoria_metrics
          - kafka

    idrac_telemetry_configurations:
      bmc_group_data_path: ""
      mysqldb_storage: "1Gi"
      oim_bmc_ips:
        oim1: ""
        oim2: ""
    ```

    !!! note

        The default value of `bmc_group_data_path` is empty. To collect iDRAC
        metrics, set it to an existing BMC CSV, for example:

        ```text
        $OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/bmc_group_data.csv
        ```

        You can specify any other absolute path accessible from the OIM.

2. Keep the `idrac_telemetry_storage` resource sections in
   `telemetry_storage_config.yml` and the `images.idrac` entries in
   `telemetry_packages.yml` aligned with the images available to the cluster.

3. Run the Telemetry precheck. Choose one execution method; do not run both
   commands for the same operation.

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry --tags precheck
        ```

        The wrapper loads the installed Omnia environment and activates the
        configured virtual environment before running the Telemetry playbook.

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source /opt/omnia/activate-omnia.sh
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml --tags precheck
        ```

        If `OMNIA_DATA_PATH` uses a nondefault value, activate
        `<OMNIA_DATA_PATH>/activate-omnia.sh` instead.

4. Validate the Telemetry inputs and collect the required credentials:

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

    Enter the requested BMC and MySQL credentials when the credential workflow
    finds an empty value.

5. Deploy the enabled Telemetry configuration:

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

6. To run validation and deployment in one invocation, omit the tag:

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

    The untagged flow does not run the opt-in precheck. Run step 3 separately
    when an environment precheck is required.

## Verification

### Verify iDRAC Telemetry pods

Verify that the iDRAC Telemetry pod is running:

```bash title="Run on: Kubernetes control plane"
kubectl get pods -n telemetry
```

![iDRAC Telemetry pods](../../assets/images/idrac_telemetry_pods.png)

!!! note

    The `idrac-telemetry-0` pod is a StatefulSet that collects telemetry data
    from all management nodes (`oim`, `service_kube_control_plane_x86_64`,
    `service_kube_node_x86_64`, `login_node_x86_64`, and others). The number of
    `idrac-telemetry` pod replicas is determined by the number of unique
    `PARENT_SERVICE_TAG` values in the mapping file. Each replica collects
    telemetry from the iDRAC interfaces of nodes that share the same parent
    service tag.

### Verify iDRAC messages in Kafka

To verify that iDRAC Telemetry data is being successfully published to the
`idrac` Kafka topic:

1. Log in to the Service Kubernetes control plane.

2. List the Telemetry services to identify the external IP of the
   `bridge-bridge-lb` service:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get svc -n telemetry
    ```

    ![Telemetry services](../../assets/images/telemetry_services_kafka_lb.png)

3. Set the required variables:

    ```bash title="Run on: Kubernetes control plane"
    KAFKA_LB_IP=<external IP of bridge-bridge-lb service>
    TOPIC=idrac
    GROUP=idrac-consumer-group
    INSTANCE=idrac-consumer-1
    ```

4. Create a Kafka consumer:

    ```bash title="Run on: Kubernetes control plane"
    curl -X POST "http://$KAFKA_LB_IP:8080/consumers/$GROUP" \
      -H 'content-type: application/vnd.kafka.v2+json' \
      -d '{
            "name": "idrac-consumer-1",
            "format": "json",
            "auto.offset.reset": "earliest"
          }'
    ```

5. View the list of configured iDRAC Kafka topics:

    ```bash title="Run on: Kubernetes control plane"
    curl -s -X GET "http://$KAFKA_LB_IP:8080/topics" | jq '.'
    ```

6. Subscribe the consumer to the Telemetry topic:

    ```bash title="Run on: Kubernetes control plane"
    curl -X POST "http://$KAFKA_LB_IP:8080/consumers/$GROUP/instances/$INSTANCE/subscription" \
      -H 'content-type: application/vnd.kafka.v2+json' \
      -d "{\"topics\": [\"$TOPIC\"]}"
    ```

7. Consume messages from the topic:

    ```bash title="Run on: Kubernetes control plane"
    while true; do
      curl -X GET "http://$KAFKA_LB_IP:8080/consumers/$GROUP/instances/$INSTANCE/records" \
        -H 'accept: application/vnd.kafka.json.v2+json' | jq '.'
      sleep 2
    done
    ```

If Telemetry metrics are collected correctly, the output contains
JSON-formatted iDRAC Telemetry records.

### Verify TLS configuration

#### VictoriaMetrics TLS

Run the current `external_victoria` utility through the Telemetry playbook:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run telemetry --tags external_victoria
```

The CLI runs `src/telemetry/playbooks/telemetry.yml`, which imports
`playbooks/utils/external_victoria_connect.yml`. Confirm that the following
file contains the VictoriaMetrics endpoints and TLS configuration:

```text
$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_victoria/external_victoria_connect_details.yml
```

When TLS is enabled, confirm that `ca.crt` exists in the same directory. The
utility fails if the VictoriaMetrics pods or LoadBalancer endpoints are not
available.

#### Kafka TLS

Run the current `external_kafka` utility through the Telemetry playbook:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run telemetry --tags external_kafka
```

The CLI runs `src/telemetry/playbooks/telemetry.yml`, which imports
`playbooks/utils/external_kafka_connect.yml`. Confirm that the following
directory contains the connection details and the `ca.crt`, `user.crt`, and
`user.key` TLS files:

```text
$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_kafka/
```

The utility fails if the Kafka pods, native Kafka endpoint, HTTP Bridge
endpoint, or TLS material are unavailable.

### View iDRAC metrics in the VictoriaMetrics UI (VMUI)

Use the VMUI to validate that iDRAC Telemetry data is being collected and
stored successfully.

1. Verify that the VictoriaMetrics pods are running:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get pods -n telemetry -o wide | grep vm
    ```

    ![VictoriaMetrics pods](../../assets/images/victoria_metrics_pod_cluster_mode.png)

2. Verify that the VictoriaMetrics service is running:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get service -n telemetry | grep vm
    ```

    ![VictoriaMetrics service](../../assets/images/victoria_metrics_service_cluster.png)

3. Read `victoria_metrics.endpoints.vmselect.ui_url` from:

    ```text
    $OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_victoria/external_victoria_connect_details.yml
    ```

4. Access the VMUI in a web browser:

    ```text
    <scheme>://<external vmselect loadbalancer IP>:8481/select/0/vmui
    ```

    The generated URL uses `https` when the `victoria-tls-certs` Secret exists
    and `http` otherwise.

5. Verify that metrics are reaching VictoriaMetrics by running the following
   query in the VMUI:

    ```promql
    {__name__=~"PowerEdge_.*"}
    ```

    ![VictoriaMetrics VMUI](../../assets/images/victoria_metrics_vmui.png)

### Access the MySQL database

After `./omnia.sh --run telemetry --tags deploy` completes for the service
cluster, you can check the MySQL database inside the `mysqldb` container. This
command invokes `src/telemetry/playbooks/telemetry.yml`.

1. Get the names of all the iDRAC Telemetry pods:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get pods -n telemetry -l app=idrac-telemetry
    ```

    !!! note

        The `idrac-telemetry-0` pod is always responsible for collecting the
        telemetry data of the management nodes (`oim`,
        `service_kube_control_plane_x86_64`, `service_kube_node_x86_64`,
        `login_node_x86_64`, and others).

2. Connect to MySQL:

    ```bash title="Run on: Kubernetes control plane"
    kubectl exec -it -n telemetry <iDRAC_telemetry_pod_name> -c mysqldb -- mysql -u <MYSQL_USER> -p
    ```

    When prompted, enter the MySQL password.

3. At the MySQL prompt, select the `idrac_telemetrydb` database:

    ```sql
    USE idrac_telemetrydb;
    ```

4. Access the `services` table:

    ```sql
    SELECT * FROM services;
    ```

## Lifecycle and cleanup

Setting `telemetry_sources.idrac.metrics_enabled: false` and running Telemetry
deployment scales the `idrac-telemetry` StatefulSet to zero replicas. The MySQL
PVC is preserved so the service inventory remains available when iDRAC
telemetry is enabled again.

To remove only the iDRAC Telemetry resources while preserving the MySQL PVC:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run telemetry --tags cleanup_idrac
```

Delete the MySQL PVC only when a complete iDRAC telemetry data reset is
intended:

```bash title="Run on: OIM"
./omnia.sh --run telemetry --tags cleanup_idrac -e Delete_volume=true
```

!!! warning

    `Delete_volume=true` permanently removes the MySQL service inventory.

## Next steps

- Use [Export Kafka Connection Details](configure_external_kafka.md) when an
  external client needs the Kafka endpoint and certificates.

## Troubleshooting

- **The configuration is rejected:** Ensure iDRAC is enabled, both targets are
  present, and `mysqldb_storage` is not empty.
- **A BMC is listed as invalid:** Confirm the common BMC credentials, Redfish
  availability, required firmware, and Datacenter license.
- **A BMC is unreachable:** Restore network access from a service worker or the
  control-plane VIP. The workflow selects the first service worker and retries
  the second service worker, when present, before it falls back to the VIP. The
  Telemetry source validates reachability but does not configure site VLANs or
  routes.
- **Kafka or VictoriaMetrics is missing:** Confirm the corresponding sink is
  deployed; both are required by the iDRAC role.
- **MySQL is not ready:** Inspect the `mysqldb` container and the
  `cleanup-mysql-locks` init container logs.
