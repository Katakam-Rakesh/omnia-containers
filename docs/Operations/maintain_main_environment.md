# Maintain the Main environment

## Overview

Use Main maintenance commands to audit dependency-version declarations or to
remove the installed Omnia execution environment. Dependency auditing is
independent of cleanup.

Main provides two alternative cleanup modes:

| Mode | Result |
|---|---|
| `--cleanup` | Removes the installed environment while preserving runtime input, output, and log data under `OMNIA_DATA_PATH`. |
| `--cleanup --all` | Performs a guarded full reset and removes the complete configured `OMNIA_DATA_PATH`. |

Choose one cleanup mode. Do not run normal cleanup before full cleanup.

## Prerequisites

- Run the commands from the `src/main` directory in an Omnia source checkout.
- Use an account with privileges to remove the installed system environment,
  virtual environment, dependency cache, and applicable runtime data.
- Before either cleanup mode, confirm `OMNIA_DATA_PATH` and `OMNIA_VENV_PATH`
  in `/etc/omnia/omnia.env`.
- Run all required domain cleanup workflows before removing the shared virtual
  environment. Domain cleanup requires that environment. See
  [Clean up the OIM](oim_cleanup.md).
- Before a full reset, copy all required content out of `OMNIA_DATA_PATH`.
  The reset also removes data that a domain cleanup preserved beneath that
  path.

## Audit dependency declarations

The dependency audit does not require an installed Omnia environment. Run:

```bash title="Run on: OIM host"
cd <OMNIA_SOURCE_PATH>/src/main
./omnia.sh --check-deps
```

The command scans every known domain's `requirements.txt` and
`requirements.yml`. It exits with a nonzero status when the same Python
package or Ansible Galaxy collection has different version specifications
across domains.

Resolve reported mismatches in the controlled source before creating or
updating the shared environment. Customers using an unmodified release should
contact their support provider before changing supplied requirement files.

## Remove the installed environment and preserve runtime data

Use normal cleanup when runtime input, output, and logs must remain. If any
deployed domain might need cleanup later, clean that domain first; normal Main
cleanup removes the virtual environment required by its playbook.

Load the installed path values into the current shell so they remain available
for verification, and then run cleanup:

```bash title="Run on: OIM host"
source /etc/profile.d/omnia-env.sh
cd <OMNIA_SOURCE_PATH>/src/main
./omnia.sh --cleanup
```

Review the paths displayed by the command. Type exactly `yes` to remove the
virtual environment, installed environment files, activation helper,
dependency cache, and command helper files. Runtime input, output, and logs
under `OMNIA_DATA_PATH` are preserved.

## Perform a guarded full reset

Complete the domain cleanup procedure before requesting a full reset:

1. Run the applicable domain cleanup commands in reverse dependency order as
   described in [Clean up the OIM](oim_cleanup.md#clean-up-deployed-domains).
2. Resolve every domain cleanup failure before continuing.
3. Copy any remaining data that must be retained out of `OMNIA_DATA_PATH`.
4. Load the installed path values and run the full reset:

    ```bash title="Run on: OIM host"
    source /etc/profile.d/omnia-env.sh
    cd <OMNIA_SOURCE_PATH>/src/main
    ./omnia.sh --cleanup --all
    ```

Before displaying the confirmation prompt, the command verifies that
`OMNIA_DATA_PATH` is safe and contains no deployed, generated, nonempty, or
unrecognized domain state. If the check reports a path, no files are removed.
Run the matching domain cleanup or remove the reported retained path after
confirming it is no longer required, and then retry.

After the safety check passes, review every displayed path and type exactly
`yes`. The command removes the installed environment and the complete
configured `OMNIA_DATA_PATH`.

!!! warning

    `--skip-approval` is intended only for trusted unattended automation. It
    skips the confirmation prompt but does not bypass either full-reset safety
    check.

## Verification

After normal cleanup, verify that the installed environment is absent and the
runtime data path remains:

```bash title="Run on: OIM host"
test ! -e "$OMNIA_VENV_PATH"
test ! -e /etc/omnia/omnia.env
test ! -e /etc/profile.d/omnia-env.sh
test ! -e "$OMNIA_DATA_PATH/activate-omnia.sh"
test ! -e "$OMNIA_DATA_PATH/.data/deps-cache"
test -d "$OMNIA_DATA_PATH"
```

After `--cleanup --all`, verify that the configured data path is absent:

```bash title="Run on: OIM host"
test ! -e "$OMNIA_DATA_PATH"
```

These commands produce no output when every check succeeds. The path variables
remain available because the installed environment was sourced before cleanup.

## Next steps

- After cleanup, [set up the OIM](../HowTo/main/setup_oim.md) again when a new
  environment is required.
- Reconfigure and deploy only the required domains in dependency order.

## Troubleshooting

- **The dependency audit exits nonzero**: Review the reported domains and
  version specifications. Do not continue with conflicting requirements.
- **The installed profile is missing before cleanup**: Read the configured
  paths from `/etc/omnia/omnia.env`. If both installed files are missing,
  explicitly set the original `OMNIA_DATA_PATH` and `OMNIA_VENV_PATH` before
  cleanup so the default paths are not used unintentionally.
- **Full cleanup reports incomplete domain cleanup**: No files were removed.
  Run the cleanup tag for the reported domain. If the domain cleanup completed
  but retained a status, log, output, or backup file, preserve it if required,
  remove the reported path, and retry.
- **Full cleanup rejects the configured data path**: Do not bypass the check.
  `OMNIA_DATA_PATH` cannot be a broad system directory or contain a symbolic
  link. Correct the installed configuration and verify the resolved path.
- **Cleanup is cancelled**: The interactive prompt accepts only the exact
  response `yes`.
- **Files remain after cleanup**: Confirm that the account can remove every
  displayed system, virtual-environment, and data path. Main full cleanup
  removes `OMNIA_DATA_PATH`; custom component data roots or external storage
  must be handled by their owning domain cleanup workflow.
