# Orchestrator Domain Contract

**Deployment module**: Orchestrator | **CLI identifier**: `orchestrator`

## Phase input requirements

Orchestrator phase tags select only that phase; they do not run earlier
prerequisite phases automatically.

| Phase | Catalog | `repo_status.yml` | `build_status.yml` | Prior state |
|---|---|---|---|---|
| `validate` | No | No | No | Staged Orchestrator YAML inputs |
| `precheck` | Yes | Successful | Successful | Current PXE mapping and reachable image artifacts |
| `credentials` | Yes | No | No | Applicable credential values |
| `prepare` | Yes | No | No | Current PXE mapping |
| `deploy` | Yes | No | No | Completed `prepare`, including stored credentials |
| `provision` | Yes | Successful | Successful | Successful `precheck` and `prepare`; healthy deployed services |
| `execute` | Yes | Successful | Successful | Same as `provision`; BMC access when PXE is enabled |
| `validate-deployment` | Yes | No | No | Deployed services, current network specification and PXE mapping, generated functional-group state, `$OMNIA_DATA_PATH/openchami/configs_vars.yaml`, and stored authentication state |
| `pxeboot` | No | Successful | No | Completed provisioning, stored BMC credentials, and reachable mapped iDRACs |

Cleanup and credential cleanup do not require upstream status files. Upgrade
requires a supported deployed source version and successful
`repo_status.yml`; rollback is unavailable in this release.

The standalone `validate-deployment` preamble reloads persisted cluster state,
authenticates to OpenCHAMI, and can ensure the cluster-hostname entry in
`/etc/hosts`. Treat it as a readiness operation, not as a purely read-only
inspection.

## Upstream domain contracts

| Producer | Output consumed by Orchestrator | Required contract |
|---|---|---|
| Repository Manager | `repo_status.yml` | Precheck, provisioning, execute, and PXE flows validate `overall_status: success`, OS metadata, repository mappings, and the Pulp certificate path; the certificate must exist. |
| Image Build Manager | `build_status.yml` | Precheck, provisioning, and execute flows require `overall_status: success` and a usable S3 endpoint. Functional-group image records supply the boot artifacts. A standalone `pxeboot` run does not read this file. |
| Discovery or an administrator | `pxe_mapping_file.csv` | Node data must use the mapping columns below. Discovery output must be reviewed and staged for Orchestrator; the handoff is not automatic. |

### Contract locations and structure samples

| Contract | Producer output | Structure sample |
|---|---|---|
| `repo_status.yml` | `$REPO_MANAGER_DATA_PATH/output/$OMNIA_PROJECT_NAME/repo_status.yml` | [Repository Manager contract](repo_manager_contract.md) |
| `build_status.yml` | `$IMAGE_BUILD_MANAGER_DATA_PATH/output/$OMNIA_PROJECT_NAME/build_status.yml` | [Image Build Manager contract](image_build_manager_contract.md) |
| `pxe_mapping_file.csv` | `$DISCOVERY_DATA_PATH/output/$OMNIA_PROJECT_NAME/bmc_pxe_mapping_file.csv` | [PXE mapping structure](../SampleFiles/pxe_mapping_file.md) |

Custom Repo Manager and Image Build Manager output paths can be set in
`orchestrator_config.yml`. Discovery output must be reviewed and copied to
`$ORCHESTRATOR_DATA_PATH/input/$OMNIA_PROJECT_NAME/pxe_mapping_file.csv`.
Generated producer outputs remain authoritative; the linked files show the
structures expected by Orchestrator.

Each component data path defaults to its directory under `$OMNIA_DATA_PATH`.
For example, `ORCHESTRATOR_DATA_PATH` defaults to
`$OMNIA_DATA_PATH/orchestrator`. A component-specific value is authoritative
when configured.

The PXE mapping contract is:

```text
FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
```

`FUNCTIONAL_GROUP_NAME`, `GROUP_NAME`, `HOSTNAME`, `ADMIN_MAC`, and `ADMIN_IP`
are required for each node. `SERVICE_TAG` is optional, but a nonempty value
must be alphanumeric and unique. Physical PXE operations also use the BMC
fields. `IB_NIC_NAME` and `IB_IP` are supplied together for nodes that use
InfiniBand. Keep all 11 columns in the exact order shown, including columns
whose values are optional.

## Output contract

Customer-readable project outputs are written under:

```text
$ORCHESTRATOR_DATA_PATH/output/$OMNIA_PROJECT_NAME/
```

`provisioning_report.yml`, `orchestrator_status.yml`, `pxeboot_status.yml`,
and `failed_nodes.json` use schema version `1.0`.

| Output | Purpose |
|---|---|
| `orchestrator_status.yml` | Stable aggregate containing the provisioning and PXE phase states. |
| `provisioning_report.yml` | Expected and registered node counts, missing nodes, and missing boot or metadata configurations. |
| `orchestrator_inventory.yaml` | Generated Ansible inventory for mapped nodes. `kube_vip_group` is included only when a mapped functional group starts with `service_kube_` and a Kubernetes VIP is available. |
| `bmc_group_data.csv` | BMC inventory generated for downstream iDRAC telemetry. It includes an OIM row only when `Networks.admin_network.primary_oim_bmc_ip` is set. |
| `failed_nodes.json` | Per-node failures produced by the iDRAC PXE-boot and registration flow. |
| `pxeboot_status.yml` | PXE initiation and optional node-verification results for every selected node. |
| `orchestrator_state.yml` | Persisted feature flags used by subsequent and standalone flows. |

Orchestrator also writes the shared generated file:

```text
$ORCHESTRATOR_DATA_PATH/output/$OMNIA_PROJECT_NAME/.data/functional_groups_config.yml
```

This file is derived from `pxe_mapping_file.csv` and is consumed by inventory
generation, OpenCHAMI configuration, Slurm and Kubernetes provisioning, and
validation roles.

Its generated structure is:

```yaml
groups:
  grp0:
    parent: ""
  grp1:
    parent: "ABFL82"

functional_groups:
  - name: "slurm_control_node_rhel_10_0_x86_64"
    cluster_name: "slurm_cluster"
    group:
      - grp0
  - name: "slurm_node_rhel_10_0_x86_64"
    cluster_name: "slurm_cluster"
    group:
      - grp1
```

`groups` maps each PXE `GROUP_NAME` to its optional parent service tag.
`functional_groups[].group` contains group names, not per-node inventory
records. Node records remain in the PXE mapping and generated
`orchestrator_inventory.yaml`.

### `orchestrator_status.yml`

`orchestrator_status.yml` uses the same schema for the provisioning and PXE
phases. After provisioning, `last_completed_phase` is `provisioning`, and
`phases.pxeboot.status` is `not_run`. After PXE boot,
`last_completed_phase` is `pxeboot`; `phases.provisioning` retains the
available provisioning result only when that report's `inventory_source`
exactly matches the active PXE inventory. A custom retry or subset inventory
therefore records the provisioning phase as `not_run` and per-node
provisioning state as `unknown`. `phases.pxeboot` records the PXE result. The
top-level node and count fields describe the latest completed phase.

After PXE boot, `overall_status` is `failed` when the PXE phase fails or the
available provisioning report contains missing nodes. The `artifacts` section
identifies `provisioning_report.yml`, `pxeboot_status.yml`, and
`failed_nodes.json`.

Provisioning `overall_status` is determined by missing SMD nodes. Missing boot
configurations, metadata, admin interfaces, or hostname assignments remain
visible in their report arrays but do not independently change that status.

When node-registration verification is enabled, Orchestrator connects to each
node's admin IP through passwordless root SSH. It derives the boot time from
`/proc/uptime`, requires it to be newer than the current PXE operation, and
requires `cloud-init status --long` to report `done`. Metadata Service
phone-home callbacks are not used. Nodes that fail the iDRAC restart phase are
retained in `failed_nodes.json` and excluded from registration polling.

### OpenCHAMI runtime artifacts

The active category-based provisioning workflow creates internal files under:

```text
$OMNIA_DATA_PATH/openchami/workdir/nodes/
```

| Runtime artifact | Purpose |
|---|---|
| `nodes_<category>.yaml` | Supplies the category-specific node records registered in SMD. Categories are `kubernetes`, `slurm`, `os`, and `custom`. |
| `hostname_<category>.yaml` | Supplies category-specific xname-to-hostname assignments to metadata-service. |
| `groups-<functional_group>.yml` | Supplies functional-group membership registered in SMD. |
| `groups-common-<name>.yml` | Supplies common metadata-service groups such as SSH, chrony, and node registration. |

These files are internal working data, not customer-readable project outputs.
The generic `nodes.yaml`, `groups.yaml`, and `hostname.yaml` names belong to the
older non-category workflow and are not the primary artifacts generated by the
current provisioning playbooks.

### Service outputs

Orchestrator also creates runtime state through OpenCHAMI and the selected
cluster roles. These include SMD node and group records, boot-service
configurations, metadata-service cloud-init resources, OpenCHAMI services, and
the selected Slurm or Kubernetes deployment. These service resources are not
represented by generated `slurm_config.yml` or `kubernetes_config.yml` files
in the project output directory.

## Lifecycle contract

### Cleanup

The top-level `cleanup` tag removes all enabled components and Orchestrator
credentials. The `cleanup_credentials` tag limits the operation to credential
artifacts, while `cleanup,cleanup_credentials` explicitly removes both the
enabled components and credentials. Although source comments mention a
`cleanup_credentials=false` extra variable, the current cleanup implementation
does not consume it. Retain credentials by running the standalone cleanup
playbook with explicit component tags that omit `cleanup_credentials`, or by
using an approved secure backup and restore procedure.

Component tags are not accepted by the top-level Orchestrator playbook. Run
`playbooks/cleanup/cleanup_orchestrator.yml` directly for `openchami`,
`openldap`, `slurm`, `k8s`, `storage_mounts`, or `artifacts`. Slurm and
Kubernetes cleanup select storage unmounting as a dependency. When their
shared data is reachable through a mounted share or a local NFS export, the
workflow can permanently delete managed directories. `DRY_RUN=true` uses
Ansible check mode. Destructive execution requires the exact interactive
response `yes`, unless `SKIP_APPROVAL=true` explicitly enables non-interactive
cleanup.

### Upgrade and rollback

The `upgrade` tag runs the current OpenCHAMI and OpenLDAP upgrade workflows. The
OpenCHAMI workflow detects the installed package, targets the `0.1.7-1` to
`0.2.0-1` migration, creates a timestamped backup, removes legacy services when
present, pulls configured images, restarts services, and performs health
checks. It does not install or verify a target OpenCHAMI RPM. The OpenLDAP
workflow pulls image tag `1.2` and restarts an existing `omnia_auth` service,
but does not rewrite a fixed-image Quadlet. Operators must verify the installed
package, configured image, and running service versions after either workflow.
OpenLDAP is skipped when its container is absent.

The `rollback` tag is reserved. Both current component rollback playbooks
intentionally fail with `ROLLBACK NOT SUPPORTED`; the OpenCHAMI upgrade backup
does not provide an automated rollback path.

## Related documentation

- [Orchestrator](../../HowTo/orchestrator/index.md)
- [Provision nodes](../../HowTo/orchestrator/provision_nodes.md)
- [Upgrade Orchestrator](../../HowTo/orchestrator/upgrade_orchestrator.md)
- [Clean up Orchestrator](../../HowTo/orchestrator/cleanup_orchestrator.md)
- [Repository Manager contract](repo_manager_contract.md)
- [Image Build Manager contract](image_build_manager_contract.md)
- [PXE mapping file](../SampleFiles/pxe_mapping_file.md)
