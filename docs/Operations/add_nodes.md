# Add Nodes

## Overview

Orchestrator treats the current `pxe_mapping_file.csv` as the desired node
inventory. To add nodes, append valid rows, rerun provisioning, and PXE boot
only the new physical servers by passing a custom inventory CSV to the
source-supported `pxeboot_inventory` extra variable.

The provisioning pass handles every mapped category: Kubernetes, Slurm and
login, OS-only, and custom functional groups. It re-registers category nodes in
SMD, refreshes functional-group boot and metadata configuration, prepares the
configured bolt-ons, and regenerates reports and inventories.

## Prerequisites

- Complete [Provision Nodes](../HowTo/orchestrator/provision_nodes.md) for the existing cluster.
- Discover the new hardware and collect every required mapping value. Physical
  nodes require reachable iDRAC `BMC_IP` values and valid BMC credentials.
- Ensure each new service tag, hostname, admin IP, and MAC address is correct
  and does not duplicate an existing entry.
- Ensure Image Build Manager's successful `build_status.yml` has an image for
  every new functional-group name. If a new functional group was added to the
  catalog, rerun Repo Manager and Image Build Manager first.
- Confirm that the admin IPs belong to the admin networking configured in
  `network_spec.yml`.

## Procedure

1. Resolve the active project paths:

    ```bash title="Run on: OIM"
    source /etc/profile.d/omnia-env.sh
    source "$OMNIA_DATA_PATH/activate-omnia.sh"
    orchestrator_path="${ORCHESTRATOR_DATA_PATH:-${OMNIA_DATA_PATH}/orchestrator}"
    discovery_path="${OMNIA_DATA_PATH}/discovery"
    orchestrator_input="$orchestrator_path/input/$OMNIA_PROJECT_NAME"
    orchestrator_output="$orchestrator_path/output/$OMNIA_PROJECT_NAME"
    discovery_output="$discovery_path/output/$OMNIA_PROJECT_NAME"
    ```

2. Append the new rows to the mapping configured by
   `pxe_mapping_file_path`. When that value is empty, the mapping is
   `$orchestrator_input/pxe_mapping_file.csv`. Preserve the case-sensitive
   header and all existing rows.

    ```text title="Required CSV header"
    FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
    ```

   If Discovery produced the new hardware mapping, review
   `$discovery_output/bmc_pxe_mapping_file.csv` and merge the approved new rows
   into the Orchestrator mapping. Discovery does not update the Orchestrator
   input automatically; do not replace retained rows without reviewing the
   difference.

3. Create a second CSV containing the same header but only the new physical
   nodes. This subset is used only for the PXE-boot phase; the primary mapping
   remains the complete desired inventory.

4. Validate the updated primary mapping, run prechecks, and run provisioning:

    ```bash title="Run on: OIM"
    cd <OMNIA_SOURCE_PATH>/src/main
    ./omnia.sh --run orchestrator --tags validate
    ./omnia.sh --run orchestrator --tags precheck
    ./omnia.sh --run orchestrator --tags provision
    ```

5. PXE boot only the new nodes by supplying the subset CSV:

    ```bash title="Run on: OIM"
    ./omnia.sh --run orchestrator --tags pxeboot \
      -e pxeboot_inventory=/absolute/path/to/new_nodes.csv
    ```

   The source reads the subset to build the BMC group, sets the configured boot
   source through Redfish, restarts the nodes, and waits for their admin IPs to
   become reachable after the PXE trigger. It does not reboot rows omitted from
   the custom inventory.

   For virtual machines or environments without iDRAC, keep
   `enable_pxe_boot: false`; the source skips the iDRAC PXE flow.

## Verification

Check the full provisioning result and the new-node PXE result separately:

```bash title="Run on: OIM"
cat "$orchestrator_output/provisioning_report.yml"
cat "$orchestrator_output/pxeboot_status.yml"
cat "$orchestrator_output/failed_nodes.json"
```

With a custom PXE inventory, `pxeboot_status.yml` reports only that subset and
records the inventory source. `failed_nodes.json` is empty when all new nodes
complete both iDRAC PXE boot and node registration.

For Slurm additions, confirm the new compute node from a controller:

```bash title="Run on: Slurm controller"
scontrol show node <new-compute-hostname>
sinfo
```

For Kubernetes additions, check the cluster from the first control-plane node:

```bash title="Run on: first Kubernetes control-plane node"
kubectl get nodes -o wide
kubectl get pods --all-namespaces -o wide
```

## Next steps

- Retain the updated primary mapping as the desired inventory for future runs.
- Use [Remove Slurm Nodes](remove_slurm_nodes.md) when decommissioning Slurm compute
  nodes.
- Adjust node-registration timing in `set_pxe_boot_config.yml` if the added
  hardware consistently needs a longer boot window.

## Troubleshooting

**Provisioning reports a missing image**

The functional-group name must have a matching entry in Image Build Manager's
successful `build_status.yml`. Build the missing image and rerun `precheck` and
`provision`.

**The PXE phase would include existing nodes**

Do not run `--tags pxeboot` against the complete mapping for this operation.
Pass a same-format CSV containing only the new rows with
`-e pxeboot_inventory=/absolute/path/to/new_nodes.csv`.

**A new node fails PXE boot or node registration**

Use `failure_stage` in `pxeboot_status.yml` or `failed_nodes.json` to separate
an iDRAC failure from a node-registration timeout. Check BMC connectivity for
`pxe_boot`; for `node_registration`, check admin-network SSH reachability,
metadata-service, and whether the node actually restarted after the PXE
trigger.
