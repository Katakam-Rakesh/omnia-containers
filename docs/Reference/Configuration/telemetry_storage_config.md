# telemetry_storage_config.yml

This file configures resource requests, resource limits, replica counts, and
persistent volume sizes for telemetry storage and processing components.

## Parameter Reference

### Telemetry Storage Configuration Parameters

--8<-- "html/telemetry_storage_config.html"

## Usage example

```yaml title="File: /opt/omnia/telemetry/input/project_default/telemetry_storage_config.yml"
---
victoria_cluster_storage:
  vmstorage:
    replicas: 3
    resources:
      requests:
        memory: "1Gi"
        cpu: "250m"
      limits:
        memory: "2Gi"
        cpu: "1000m"
  vminsert:
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  vmselect:
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  vmagent:
    replicas: 2
    resources:
      requests:
        memory: "128Mi"
        cpu: "50m"
      limits:
        memory: "512Mi"
        cpu: "250m"

victoria_logs_cluster_storage:
  vlstorage:
    replicas: 3
    resources:
      requests:
        memory: "512Mi"
        cpu: "100m"
      limits:
        memory: "1Gi"
        cpu: "500m"
  vlinsert:
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  vlselect:
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  vlagent:
    replicas: 2
    pvc_size: "5Gi"
    resources:
      requests:
        memory: "64Mi"
        cpu: "25m"
      limits:
        memory: "256Mi"
        cpu: "100m"

vector_storage:
  ldms:
    replicas: 2
    resources:
      requests:
        memory: "128Mi"
        cpu: "50m"
      limits:
        memory: "256Mi"
        cpu: "250m"
  ome:
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  vlagent_vector:
    replicas: 2
    pvc_size: "5Gi"
    resources:
      requests:
        memory: "128Mi"
        cpu: "50m"
      limits:
        memory: "256Mi"
        cpu: "250m"
  vmagent_vector:
    replicas: 2
    pvc_size: "5Gi"
    resources:
      requests:
        memory: "128Mi"
        cpu: "50m"
      limits:
        memory: "256Mi"
        cpu: "250m"

csi_volume_exporter_storage:
  resources:
    requests:
      cpu: "50m"
      memory: "64Mi"
    limits:
      cpu: "200m"
      memory: "256Mi"

csm_metrics_powerscale_storage:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

idrac_telemetry_storage:
  mysqldb:
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  activemq:
    resources:
      requests:
        cpu: "100m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "1536Mi"
  receiver:
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
  kafka_pump:
    resources:
      requests:
        cpu: "50m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "512Mi"
  victoria_pump:
    resources:
      requests:
        cpu: "50m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "512Mi"

kafka_storage:
  kafka:
    resources:
      requests:
        memory: "512Mi"
        cpu: "200m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
  entity_operator:
    user_operator:
      resources:
        requests:
          memory: "512Mi"
          cpu: "200m"
        limits:
          memory: "512Mi"
          cpu: "1000m"
```

!!! info

    - [Telemetry Configuration](telemetry_config.md) -- Telemetry sources, bridges, sinks, and component-specific settings.
    - [Telemetry Packages](telemetry_packages.md) -- Package sources, images, charts, repositories, and Python modules.
