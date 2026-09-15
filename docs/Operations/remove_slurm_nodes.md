# Remove Slurm Compute Nodes

## Overview

The source implements node removal in the Slurm configuration path. It compares
the compute nodes in the generated mapping with the nodes in the `normal`
partition, detects omitted nodes, protects nodes with active jobs through an
interactive choice, updates the generated Slurm configuration, removes the
node's shared configuration directory, and runs `scontrol reconfigure` when the
controller is available.

This procedure is limited to Slurm compute nodes. The source requires a Slurm
controller to remain in the mapping and does not implement equivalent removal
logic for Kubernetes, login, controller, OS-only, or custom nodes.

## Prerequisites

- Complete [Deploy Slurm](../HowTo/orchestrator/deploy_slurm.md) and ensure the controller is
  reachable on TCP port 22.
- Identify the exact Slurm compute hostnames to remove. The role compares them
  against the compute list built from `slurm_node_` functional groups.
- Run the command interactively if any target node may have active jobs. The
  source prompts for `A` or `F`.
- Keep at least one `slurm_control_node_...` entry in the mapping; provisioning
  fails when the controller list is empty.

## Procedure

Resolve the active project paths before changing the inventory:

```bash title="Run on: OIM"
source /etc/profile.d/omnia-env.sh
source "$OMNIA_DATA_PATH/activate-omnia.sh"
orchestrator_path="${ORCHESTRATOR_DATA_PATH:-${OMNIA_DATA_PATH}/orchestrator}"
orchestrator_input="$orchestrator_path/input/$OMNIA_PROJECT_NAME"
orchestrator_output="$orchestrator_path/output/$OMNIA_PROJECT_NAME"
```

### 1. Update the desired inventory

Remove only the intended `slurm_node_...` rows from the primary mapping
configured by `pxe_mapping_file_path`. When that value is empty, update
`$orchestrator_input/pxe_mapping_file.csv`. Keep all remaining cluster rows.

### 2. Validate and apply the change

```bash title="Run on: OIM"
cd <OMNIA_SOURCE_PATH>/src/main
./omnia.sh --run orchestrator --tags validate
./omnia.sh --run orchestrator --tags precheck
./omnia.sh --run orchestrator --tags provision
```

### 3. Respond to active jobs

If omitted nodes have active jobs, choose one of the source-defined actions:

- Enter `A` to remove idle omitted nodes, retain busy nodes, and stop the
  playbook with instructions to wait for or cancel their jobs before rerunning
  it.
- Enter `F` to force removal. The role runs `scancel -f -w <node>`, marks the
  node down, stops `slurmd` when the host is reachable, and deletes its Slurm
  directory from the configured share.

Before choosing force removal, review the jobs with the command displayed by
the playbook:

```bash title="Run on: Slurm controller"
squeue -w <compute-hostname>
```

If you choose `A`, wait for jobs to finish or cancel them, then rerun the
`provision` tag to complete removal.

## Verification

On the Slurm controller, confirm that the removed node is no longer configured:

```bash title="Run on: Slurm controller"
scontrol show node <removed-compute-hostname>
sinfo
```

`scontrol show node` should no longer find the removed compute node, and `sinfo`
should show only the compute nodes retained in the mapping.

Review the regenerated Orchestrator report and inventory:

```bash title="Run on: OIM"
cat "$orchestrator_output/provisioning_report.yml"
cat "$orchestrator_output/orchestrator_inventory.yaml"
```

## Next steps

- Decommission or repurpose the hardware according to your site procedure after
  Slurm no longer reports it.
- Use [Add Nodes](add_nodes.md) to add replacement compute nodes.
- Keep the primary PXE mapping synchronized with the intended cluster
  inventory before future provisioning runs.

## Troubleshooting

**Removal stops because jobs are active**

This is the expected result after choosing `A`. Wait for the displayed jobs or
cancel them on the controller, then rerun `--tags provision`.

**The controller cannot be reached**

The source checks TCP port 22 before drain and removal. Restore SSH
connectivity from the OIM to the controller and rerun provisioning.

**The node remains in Slurm after provisioning**

Confirm that the row was removed from the primary mapping, not only from a PXE
subset. Then check the controller and reconfigure it:

```bash title="Run on: Slurm controller"
systemctl status slurmctld
scontrol reconfigure
scontrol show node <removed-compute-hostname>
```

Review `/var/log/omnia/orchestrator/orchestrator.log` for failures in busy-node
detection, drain/removal, shared-directory deletion, or controller
reconfiguration.
