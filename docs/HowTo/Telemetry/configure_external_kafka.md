# Collect Telemetry Data from External Clients to Kafka

Connect an external Telemetry producer to the Kafka cluster deployed in the
Service Kubernetes cluster.

## Overview

External clients send Telemetry data to the Strimzi Kafka cluster in the
`telemetry` namespace through its native LoadBalancer endpoint. The connection
uses mutual TLS (mTLS). The `external_kafka` utility validates the Kafka
deployment and exports the project-specific endpoint, HTTP Bridge endpoint,
cluster CA certificate, and client credentials.

## Prerequisites

- Deploy Kafka through the Telemetry workflow.
- Ensure the Kafka pods in the `telemetry` namespace are Running and Ready.
- Ensure the `kafka-kafka-external-bootstrap` and `bridge-bridge-lb` services
  have LoadBalancer external IP addresses.
- Ensure the external client can reach the native Kafka LoadBalancer port.
- Ensure the OIM can reach the Kubernetes VIP over root SSH.
- Export `OMNIA_DATA_PATH` and `OMNIA_PROJECT_NAME` for the project whose
  connection details must be retrieved.
- Install OpenSSL, Java `keytool`, and Kafka command-line tools on the external
  client, or make them available in a container.

## Procedure

1. Optional: create a Kafka topic for the external producer. Save the following
   manifest as
   `$OMNIA_DATA_PATH/telemetry/input/$OMNIA_PROJECT_NAME/external-kafka-topic.yml`:

    ```yaml
    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaTopic
    metadata:
      name: my-new-topic
      namespace: telemetry
      labels:
        strimzi.io/cluster: kafka
    spec:
      partitions: 3
      replicas: 3
      topicName: my-new-topic
    ```

    Apply and verify the topic from the Kubernetes control plane:

    ```bash title="Run on: Kubernetes control plane"
    kubectl apply -f "$OMNIA_DATA_PATH/telemetry/input/$OMNIA_PROJECT_NAME/external-kafka-topic.yml"
    kubectl get kafkatopics -n telemetry
    ```

2. Retrieve the Kafka connection details. Choose one execution method; do not
   run both commands for the same operation.

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM"
        cd src/main
        ./omnia.sh --run telemetry --tags external_kafka
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM"
        source "$OMNIA_DATA_PATH/activate-omnia.sh"
        cd src/telemetry
        ansible-playbook playbooks/telemetry.yml --tags external_kafka
        ```

3. Review the project-specific output:

    ```text
    $OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_kafka/
    |-- ca.crt
    |-- user.crt
    |-- user.key
    `-- external_kafka_connect_details.yml
    ```

    Use `kafka.bootstrap_server` for a native Kafka client and
    `kafka.bridge.endpoint` for an HTTP client. In a multi-domain environment,
    always use the output below the applicable `OMNIA_PROJECT_NAME`.

4. If the external application requires a PKCS#12 certificate, create it in
   the project output directory:

    ```bash title="Run on: OIM"
    KAFKA_OUTPUT_DIR="$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_kafka"
    cd "$KAFKA_OUTPUT_DIR"
    openssl pkcs12 -export -out user.pfx -inkey user.key -in user.crt
    ```

    Each export run removes and recreates the `external_kafka` directory.
    Generate `user.pfx` after the final export and copy it to a secure location
    before rerunning the utility.

5. Optional: create Java truststore and keystore files for Kafka
   command-line tools. Replace the example passwords before production use.

    ```bash title="Run on: external Kafka client"
    KAFKA_OUTPUT_DIR="$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_kafka"
    cd "$KAFKA_OUTPUT_DIR"

    keytool -importcert -noprompt -alias CARoot -file ca.crt \
      -keystore kafka.truststore.jks -storepass changeit

    openssl pkcs12 -export -name kafka-user -in user.crt -inkey user.key \
      -out kafka-user.p12 -passout pass:changeit

    keytool -importkeystore -noprompt \
      -srckeystore kafka-user.p12 -srcstoretype PKCS12 \
      -srcstorepass changeit -destkeystore kafka.keystore.jks \
      -deststorepass changeit
    ```

6. Create `producer-mtls.properties` in `KAFKA_OUTPUT_DIR`. The `/certs`
   paths assume that this directory is mounted at `/certs` in a Kafka tools
   container:

    ```properties
    security.protocol=SSL
    ssl.truststore.location=/certs/kafka.truststore.jks
    ssl.truststore.password=changeit
    ssl.keystore.location=/certs/kafka.keystore.jks
    ssl.keystore.password=changeit
    ssl.key.password=changeit
    ```

7. Mount the project output directory in a Kafka tools container, if the
   external host does not have Kafka command-line tools:

    ```bash title="Run on: external Kafka client"
    KAFKA_OUTPUT_DIR="$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/external_kafka"
    podman run --rm -it --network host \
      -v "$KAFKA_OUTPUT_DIR:/certs:Z" \
      <kafka-client-image> bash
    ```

## Verification

1. Set the bootstrap server to the value of `kafka.bootstrap_server` in
   `external_kafka_connect_details.yml`:

    ```bash title="Run on: external Kafka client or Kafka tools container"
    KAFKA_BOOTSTRAP_SERVER=<kafka.bootstrap_server>
    TOPIC=my-new-topic
    ```

2. List the available topics over mTLS:

    ```bash title="Run on: external Kafka client or Kafka tools container"
    kafka-topics.sh --bootstrap-server "$KAFKA_BOOTSTRAP_SERVER" \
      --command-config /certs/producer-mtls.properties --list
    ```

3. Start a producer and send one or more JSON records:

    ```bash title="Run on: external Kafka client or Kafka tools container"
    kafka-console-producer.sh --bootstrap-server "$KAFKA_BOOTSTRAP_SERVER" \
      --producer.config /certs/producer-mtls.properties --topic "$TOPIC"
    ```

    ```json
    {"source":"external-client","metric":"temperature","value":24.7}
    ```

4. In another terminal, consume the records:

    ```bash title="Run on: external Kafka client or Kafka tools container"
    kafka-console-consumer.sh --bootstrap-server "$KAFKA_BOOTSTRAP_SERVER" \
      --consumer.config /certs/producer-mtls.properties \
      --topic "$TOPIC" --from-beginning
    ```

    Receiving the JSON record confirms that the topic, external endpoint, and
    mTLS client credentials are working.

## Troubleshooting

- **No Kafka pods are found:** Deploy a source that targets Kafka and rerun the
  Telemetry deployment.
- **Kafka pods are not Running or Ready:** Inspect them with
  `kubectl get pods -n telemetry -l app.kubernetes.io/name=kafka`.
- **An endpoint is empty:** Ensure both Kafka LoadBalancer services have an
  external IP address and service port.
- **TLS validation fails:** Recreate the keystore and truststore from the files
  in the current project's export directory and verify their passwords.
- **The VIP cannot be reached:** Restore root SSH access from the OIM to the
  configured Kubernetes VIP.

## Next steps

- Use the exported native endpoint and certificates to
  [configure OME](telemetry_from_ome.md).
- Protect `user.key`, `user.pfx`, and the Java keystore as client credentials.
