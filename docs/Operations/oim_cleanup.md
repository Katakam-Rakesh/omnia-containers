# Clean up the OIM

Use each domain's cleanup workflow to remove the services and artifacts that
it owns. After the required domain cleanups succeed, remove the shared Omnia
execution environment with the Main cleanup command.

Cleanup runs from the Omnia Infrastructure Manager (OIM). Keep the shared
virtual environment installed until every required domain cleanup completes.

!!! danger

    Cleanup is destructive. Depending on the selected domain and options, it
    can remove containers, repositories, images, credentials, cluster
    configuration, telemetry workloads, persistent data, backups, and
    shared-storage content. Back up all required data before continuing.

## When to use OIM cleanup

- Reset a failed or experimental deployment.
- Remove selected Omnia domains before redeployment.
- Return a lab or test OIM to a clean state.
- Remove the shared Omnia environment after domain services are removed.

## Prerequisites

- Log in to the OIM as a user with the privileges required by every selected
  cleanup workflow.
- Use the same `OMNIA_PROJECT_NAME` and resolved domain data paths used for
  deployment. In particular, preserve `ORCHESTRATOR_DATA_PATH` when
  Orchestrator used a custom root; when it is unset, Orchestrator uses
  `<OMNIA_DATA_PATH>/orchestrator`.
- Confirm that the Omnia virtual environment is available.
- Stop or drain workloads that use the services or storage being removed.
- Back up project inputs, credentials, repository content, images, telemetry
  data, databases, log archives, configuration backups, and shared-storage
  data that must be retained.
- Review the cleanup guide for each deployed domain before selecting optional
  data deletion or credential preservation behavior.

## Cleanup scope by domain

| Domain | Default cleanup scope |
|---|---|
| `build_stream` | Removes GitLab, BuildStreaM services, the watcher, PostgreSQL service, runtime artifacts, and BuildStreaM credentials. PostgreSQL data is preserved by default. |
| `telemetry` | Removes all enabled telemetry sources and sinks and deletes the stored telemetry credential file. Source-owned persistent volumes are removed; Kafka, VictoriaMetrics, and VictoriaLogs volumes are preserved by default. |
| `orchestrator` | Removes enabled OpenCHAMI, OpenLDAP, Slurm, Kubernetes, storage-mount, and generated Orchestrator resources. It prompts independently before deleting Slurm and Kubernetes shared data and removes credentials by default. |
| `discovery` | Removes the current project's Discovery output contents and credentials while preserving the output directory and other staged inputs. |
| `image_build_manager` | Removes MinIO when locally managed, the image registry, build artifacts, domain runtime data, logs, and Image Build Manager credentials. |
| `repo_manager` | Removes the Pulp deployment, Pulp data, repository integration, logs, and Repo Manager credentials. Credentials and logs are removed by default. |
| `utils` | Removes cluster-log artifacts, unattended-installation temporary files and credentials, all OIM log-backup runs, and all Slurm configuration backup runs. |

To preserve Image Build Manager services and remove only selected or all built
artifacts, use [Clean up built images](cleanup_built_images.md) instead of the
full `image_build_manager` cleanup tag.

## Clean up deployed domains

Run only the commands for domains that have deployed or generated state. Use
reverse dependency order so consumers are removed before the services they
depend on:

```bash title="Run on: OIM host"
cd <OMNIA_SOURCE_PATH>/src/main

./omnia.sh --run build_stream --tags cleanup
./omnia.sh --run telemetry --tags cleanup
./omnia.sh --run orchestrator --tags cleanup
./omnia.sh --run discovery --tags cleanup
./omnia.sh --run image_build_manager --tags cleanup
./omnia.sh --run repo_manager --tags cleanup
./omnia.sh --run utils --tags cleanup
```

Review every prompt and the final Ansible recap. Stop and correct a failure
before continuing to the next domain. Do not run Main cleanup while a domain
playbook still needs the shared virtual environment.

!!! warning

    Discovery cleanup removes all artifacts from the current project's output
    directory and removes its credential file and Vault key by default. Copy
    any mapping required by Orchestrator before cleanup. See
    [Clean up Discovery data](../HowTo/discovery/index.md#clean-up-discovery-data)
    for credential-preservation and credentials-only commands.

## Select optional cleanup behavior

### BuildStreaM

BuildStreaM stops and removes PostgreSQL but preserves its data by default. To
delete the PostgreSQL data and volumes as part of cleanup, run:

```bash title="Run on: OIM host"
./omnia.sh --run build_stream --tags cleanup -e postgres_backup=false
```

### Telemetry

Full Telemetry cleanup deletes source-owned persistent volumes and preserves
Kafka, VictoriaMetrics, and VictoriaLogs volumes by default. Delete the sink
volumes only when a complete telemetry data reset is intended:

```bash title="Run on: OIM host"
./omnia.sh --run telemetry --tags cleanup -e delete_sinks_volume=true
```

Use a component-specific cleanup tag when only one Telemetry source must be
removed. A full `cleanup` run also deletes the Telemetry credential file and
Vault key.

### Orchestrator

Orchestrator prompts independently before deleting Slurm and Kubernetes
shared data. Any response other than the exact value `yes` preserves that
component's shared data, but cleanup still unmounts its storage and removes
the corresponding `/etc/fstab` entry.

Set the choices explicitly for a noninteractive decision:

```bash title="Run on: OIM host"
./omnia.sh --run orchestrator --tags cleanup \
  -e cleanup_slurm=true -e cleanup_k8s=false
```

The current implementation does not consume `cleanup_credentials=false`,
although source comments mention it. Full cleanup therefore removes the
Orchestrator credential file and Vault key. To retain them, use the supported
standalone component flow described in
[Clean up Orchestrator](../HowTo/orchestrator/cleanup_orchestrator.md), selecting
explicit component tags and omitting `cleanup_credentials`, or preserve both
files through an approved secure backup procedure.

!!! warning

    `SKIP_APPROVAL=true` approves deletion of Slurm and Kubernetes shared data
    when `cleanup_slurm` or `cleanup_k8s` is omitted. Set both choices
    explicitly before using noninteractive cleanup.

### Repo Manager

Repo Manager removes its credentials and logs by default without prompting.
Preserve either set explicitly when required:

```bash title="Run on: OIM host"
./omnia.sh --run repo_manager --tags cleanup \
  -e cleanup_credentials=false -e cleanup_logs=false
```

### Utils

The general Utils cleanup removes all four utility artifact sets, including
OIM log backups and Slurm configuration backups, without applying retention.
Use scoped cleanup tags when any backup class must remain. Review
[Clean up Utils](../HowTo/utils/cleanup_utils.md) before running the general
command.

## Remove the shared Omnia environment

After every required domain cleanup succeeds, choose one Main cleanup mode.
For complete command behavior and verification, see
[Maintain the Main environment](maintain_main_environment.md).

To remove the installed environment while preserving domain input, output,
and log data under `OMNIA_DATA_PATH`, run:

```bash title="Run on: OIM host"
./omnia.sh --cleanup
```

To perform an explicitly approved full reset, run:

```bash title="Run on: OIM host"
./omnia.sh --cleanup --all
```

The full-reset safety check runs before the confirmation prompt. It refuses to
continue when it finds an unsafe data path or deployed, generated, nonempty,
or unrecognized domain state. No files are removed when this check fails. Run
the matching domain cleanup or review and remove the reported retained path,
then retry.

When the safety check passes, review the displayed paths and type exactly
`yes`. The command removes the installed environment and all remaining data
under `OMNIA_DATA_PATH`. Data in custom component roots or external storage is
removed only by the applicable domain cleanup workflow.

!!! warning

    Adding `--skip-approval` skips the Main confirmation prompt but does not
    bypass the full-reset safety checks.

## Verification

- Confirm that every selected domain cleanup has a successful Ansible recap.
- Verify that the intended services, containers, Kubernetes resources, and
  mounts are absent.
- Confirm that credentials, persistent volumes, shared data, and backups were
  preserved or removed according to the selected options.
- After normal Main cleanup, confirm that the virtual environment and installed
  environment files are absent and `OMNIA_DATA_PATH` remains.
- After `--cleanup --all`, confirm that the configured `OMNIA_DATA_PATH` is
  absent.

## Next steps

- Run `./omnia.sh --setup-venv` to recreate the shared environment when
  required.
- Initialize and deploy only the required domains in dependency order.
- See [Clean up Orchestrator](../HowTo/orchestrator/cleanup_orchestrator.md) for
  component-level Orchestrator cleanup.
- See [Pulp cleanup](pulp_cleanup.md) for selective repository cleanup.

## Troubleshooting

- **A domain cleanup fails**: Stop the sequence, correct the reported problem,
  and rerun that domain cleanup before removing its dependencies.
- **BuildStreaM cannot clean the GitLab host**: Confirm that the configured
  GitLab host is reachable and that the BuildStreaM credential file still
  contains the GitLab SSH password.
- **Orchestrator shared data was preserved unexpectedly**: Only the exact
  response `yes` approves interactive deletion. Rerun with reviewed
  `cleanup_slurm` and `cleanup_k8s` values when a noninteractive decision is
  required.
- **Main full cleanup reports incomplete domain cleanup**: No files were
  removed. Run the cleanup tag for the reported domain. Preserve and then
  remove any reported status, log, output, or backup file that is intentionally
  retained before retrying.
- **A custom data path remains**: Main full cleanup removes only
  `OMNIA_DATA_PATH`. Rerun the owning domain cleanup with the original project
  and component path configuration.
- **The virtual environment was removed too early**: Set up the OIM again to
  restore the shared environment, complete the required domain cleanup, and
  then rerun the selected Main cleanup mode.
