# collect_pxe.yml

This Utils input selects the provisioned nodes from which the log-collection
playbook gathers data.

## Location

```text
$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/collect_pxe.yml
```

Add admin IP addresses below the appropriate functional-group key. The current
source recognizes these groups:

- `service_kube_control_plane_x86_64`
- `service_kube_node_x86_64`
- `slurm_control_node_x86_64`
- `slurm_node_x86_64`
- `slurm_node_aarch64`
- `login_node_x86_64`
- `login_compiler_node_aarch64`

## Usage example

```yaml title="File: $OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/collect_pxe.yml"
service_kube_control_plane_x86_64:
  - 192.168.1.10

service_kube_node_x86_64:
  - 192.168.1.20

slurm_control_node_x86_64:
  - 192.168.1.30

slurm_node_x86_64:
  - 192.168.1.40
```

Leave a functional group without entries when no nodes from that group should
be included.
