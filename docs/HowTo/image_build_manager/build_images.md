# Build OS Images

Build customized OS images for diskless HPC cluster provisioning with Omnia's Image Build Manager.

## Overview

The Image Build Manager builds RHEL 10.x boot images for diskless HPC cluster
nodes. It runs on the Omnia Infrastructure Manager (OIM) and can build images
with either OpenCHAMI `image-builder` or `image-thrillhouse`.

The workflow:

1. Reads repository information from the Repo Manager's `repo_status.yml`.
2. Resolves packages from either `package_groups.yml` or a catalog JSON file.
3. Deploys a local OCI registry and, when selected, MinIO S3 storage.
4. Builds a base image and an image for each functional group.
5. Uploads the kernel, initramfs, and root filesystem artifacts to S3.
6. Writes `build_status.yml` for the provisioning workflow.

The x86_64 images are built locally on the OIM. An aarch64 image must be built
natively on a separate aarch64 host; cross-architecture builds and emulation
are not supported.

## Prerequisites

- The OIM runs RHEL 10.x and has at least 50 GB of free disk space.
- Python 3.12 or later, `ansible-core` 2.20 or later, and Podman 5.0 or later
  are installed. Module initialization installs the Python and Ansible Galaxy
  dependencies declared by the Image Build Manager.
- The Repo Manager completed successfully and its repository URLs are
  reachable from the OIM.
- For catalog mode, [select or update the catalog](../main/update_catalog.md)
  and complete Repo Manager synchronization for that catalog before building
  images.
- Run the playbooks on the OIM with privileges sufficient to create files
  under the path configured by `OMNIA_DATA_PATH` and under `/var/log/omnia`,
  manage systemd services and firewall rules, and run Podman.

Export the following environment variables in the shell used to run the
playbooks:

| Variable | Requirement |
|----------|-------------|
| `SYSTEM_ADMIN_NIC_IPV4` | Required. IPv4 address assigned to the OIM administrative NIC. |
| `SYSTEM_HOSTNAME` | Required. Short hostname of the OIM. |
| `SYSTEM_DOMAIN_NAME` | Required. Domain name of the OIM. |
| `OMNIA_DATA_PATH` | Required. Root Omnia data path; the standard value is `/opt/omnia`. |
| `OMNIA_VERSION` | Required. Included in generated image names. |
| `OMNIA_PROJECT_NAME` | Optional. Defaults to `project_default`. |
| `IMAGE_BUILD_MANAGER_DATA_PATH` | Optional. Overrides the default `<OMNIA_DATA_PATH>/image_build_manager` runtime root. |
| `CATALOG_FILE_PATH` | Required only in catalog mode. Absolute path to the catalog JSON file. |

For aarch64 images, also provide one network-reachable ARM64 host with:

- `uname -m` reporting `aarch64`.
- RHEL 10.x and Podman 5.0 or later.
- SSH port 22 reachable from the OIM.
- At least 30 GB free under `<IMAGE_BUILD_MANAGER_DATA_PATH>`.
- Access to the OIM Repo Manager, or internet access for the builder-image and
  `regctl` download fallbacks.

The workflow configures passwordless SSH from the OIM. It requests the remote
user's password when a key has not already been installed.

### Input contract

Image Build Manager reads its domain-owned inputs from
`<IMAGE_BUILD_MANAGER_DATA_PATH>/input/<OMNIA_PROJECT_NAME>/`. When
`IMAGE_BUILD_MANAGER_DATA_PATH` is unset, it resolves to
`<OMNIA_DATA_PATH>/image_build_manager`.

| Domain input | When required | Contract |
|--------------|---------------|----------|
| `image_build_config.yml` | Always | Defines the Repo Manager output path, S3 provider, build engine, package source, build controls, and optional aarch64 host. |
| `package_groups.yml` | `functional_groups_source: "config"` | Defines `os`, `os_version`, `base_packages`, and `functional_groups.<name>.packages`. Group names must end in `_x86_64` or `_aarch64` to be selected for that architecture. |
| `image_build_credentials.yml` | Prepare, credentials, build, execute, or the default untagged flow | Created and encrypted automatically with Ansible Vault. `s3_secret_key` is mandatory, `s3_access_id` is required for PowerScale, and `aarch64_ssh_password` is required when an aarch64 host is configured. |

The workflow also consumes the following upstream or external inputs. These
files are not stored in the Image Build Manager input directory.

| Upstream or external input | When required | Contract |
|----------------------------|---------------|----------|
| `repo_status.yml` | Build, execute, or the default untagged flow | Read from `repo_manager_output_path`. The default path is `<OMNIA_DATA_PATH>/repo_manager/output/<OMNIA_PROJECT_NAME>/repo_status.yml`. `overall_status` must be `success`; `repositories` must contain at least one non-empty x86_64 or aarch64 repository URL; and any configured Repo Manager certificate must exist. |
| [Catalog JSON](../main/update_catalog.md) | `functional_groups_source: "catalog"` | Read from the absolute path set in `CATALOG_FILE_PATH`. Packages are resolved through `catalog.functionallayer`, `catalog.groups`, and `catalog.packages`. Layer names beginning with `baseos` provide the base packages; other matching architecture layers become functional-group images. |

For MinIO, leave `s3_configurations.endpoint_url` empty; the endpoint is set to
`http://<SYSTEM_ADMIN_NIC_IPV4>:9000`. For PowerScale, set the provider to
`powerscale` and provide a reachable HTTP or HTTPS endpoint. The S3 bucket names
used by the workflow are fixed.

## Procedure

1. If the Image Build Manager was not initialized during OIM setup, initialize
   it from the Omnia source tree:

    !!! note "Optional after OIM setup"

        Skip this step if `./omnia.sh --setup-venv` completed without
        `--skip image_build_manager` or `--deps-only`. That setup command
        already runs `src/image_build_manager/domain-init.sh` and stages the
        Image Build Manager input files.

    ```bash title="Run on: OIM host"
    cd src/image_build_manager
    sudo ./domain-init.sh
    ```

    Initialization installs the declared dependencies, creates the runtime and
    log directories, and stages `image_build_config.yml` and
    `package_groups.yml` in
    `<OMNIA_DATA_PATH>/image_build_manager/input/<OMNIA_PROJECT_NAME>/`.

    !!! warning

        If staged input files already exist, initialization asks before
        overwriting them. Review the prompt carefully to preserve local
        configuration changes.

2. Edit the staged `image_build_config.yml`. The following example shows all
   supported configuration fields:

    ```yaml title="File: /opt/omnia/image_build_manager/input/project_default/image_build_config.yml"
    repo_manager_output_path: "/opt/omnia/repo_manager/output/project_default/repo_status.yml"

    s3_configurations:
      provider: "minio"            # minio or powerscale
      endpoint_url: ""              # empty for minio; required for powerscale

    image_build_type: "image-thrillhouse"  # image-builder or image-thrillhouse
    functional_groups_source: "catalog"    # config or catalog

    build_image:
      max_parallel: 0               # 0 builds all groups concurrently; maximum 64
      build_timeout: 7200            # 600 through 86400 seconds per build
      force_rebuild: false           # bypass the package-hash cache
      backup_s3_images: false        # copy existing compute artifacts to *_prev
      repo_ssl_verify: true          # enable repository SSL and GPG checks

    aarch64_inventory_host_ip: ""   # empty skips aarch64 builds
    aarch64_ssh_user: "root"
    ```

3. Configure one package-resolution mode:

    - For catalog mode, set `functional_groups_source: "catalog"` and export
      the default catalog path:

        ```bash title="Run on: OIM host"
        export CATALOG_FILE_PATH="${OMNIA_DATA_PATH}/catalog/catalog_rhel.json"
        ```

        !!! note

            OIM setup copies the default catalog to
            `$OMNIA_DATA_PATH/catalog/catalog_rhel.json`. To build images for
            a different supported workload, architecture, or VAST combination,
            select the appropriate catalog from:

            ```text
            <OMNIA_SOURCE_PATH>/src/main/samples/catalogs/
            ```

            OIM setup does not copy these additional catalogs. Follow
            [Select or update the catalog](../main/update_catalog.md) to select
            and copy the required catalog to the runtime catalog directory,
            and then update `CATALOG_FILE_PATH` to reference that JSON file.

    - For config mode, set `functional_groups_source: "config"` and edit the
      staged `package_groups.yml`. Packages under `base_packages` are installed
      in every image. Each key under `functional_groups` selects another image
      and supplies the additional packages for that group:

        ```yaml title="File: /opt/omnia/image_build_manager/input/project_default/package_groups.yml"
        os: "rhel"
        os_version: "10.0"

        base_packages:
          - systemd
          - kernel
          - dracut

        functional_groups:
          slurm_node_x86_64:
            packages:
              - munge
              - slurm-slurmd
        ```

    Package names must be RPM package names available from a repository listed
    in `repo_status.yml`. Do not add a separate functional-group list to
    `image_build_config.yml`; the workflow derives the groups from the selected
    package source.

4. To build aarch64 images, set `aarch64_inventory_host_ip` and
   `aarch64_ssh_user` in `image_build_config.yml`. Leave the IP empty for an
   x86_64-only build.

5. Run the environment precheck. Choose one execution method; do not run both
   commands for the same operation. Run each command from the root of the
   Omnia source tree.

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM host"
        cd src/main
        ./omnia.sh --run image_build_manager --tags precheck
        ```

        The wrapper loads the installed Omnia environment and activates the
        configured virtual environment before running the module playbook.

    === "Using ansible-playbook"

        ```bash title="Run on: OIM host"
        source /opt/omnia/activate-omnia.sh
        cd src/image_build_manager/playbooks
        ansible-playbook image_build_manager.yml --tags precheck
        ```

        If `OMNIA_DATA_PATH` uses a nondefault value, activate
        `<OMNIA_DATA_PATH>/activate-omnia.sh` instead.

    The precheck verifies that the environment matches the OIM hostname,
    domain, administrative IP, and data path. It also reports whether the
    installed Omnia environment file and Repo Manager output are present.

6. Validate the Image Build Manager inputs:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM host"
        cd src/main
        ./omnia.sh --run image_build_manager --tags validate
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM host"
        source /opt/omnia/activate-omnia.sh
        cd src/image_build_manager/playbooks
        ansible-playbook image_build_manager.yml --tags validate
        ```

    Validation checks the JSON schemas and cross-field rules for the main
    configuration and selected package source. It does not request
    credentials.

7. Prepare the build services:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM host"
        cd src/main
        ./omnia.sh --run image_build_manager --tags prepare
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM host"
        source /opt/omnia/activate-omnia.sh
        cd src/image_build_manager/playbooks
        ansible-playbook image_build_manager.yml --tags prepare
        ```

    Supply the requested S3 secret key, PowerScale access ID when applicable,
    and aarch64 SSH password when applicable. The credentials are stored in
    `image_build_credentials.yml` and encrypted with an automatically generated
    Vault key in `.image_build_credentials_key`.

    With MinIO selected, this step deploys `minio.service`, creates the `efi`
    and `boot-images` buckets, and configures their public-read policies. For
    PowerScale, local MinIO deployment is skipped. The step always deploys the
    local OCI `registry.service` used by the image build and installs `regctl`.

8. Build the images and write the output contract:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM host"
        cd src/main
        ./omnia.sh --run image_build_manager --tags build
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM host"
        source /opt/omnia/activate-omnia.sh
        cd src/image_build_manager/playbooks
        ansible-playbook image_build_manager.yml --tags build
        ```

    The build validates `repo_status.yml`, resolves packages, builds x86_64
    images on the OIM, optionally builds aarch64 images on the configured ARM
    host, uploads the artifacts, and writes `build_status.yml`.

    To run preparation and the build in one invocation, omit the tag:

    === "Using omnia.sh (recommended)"

        ```bash title="Run on: OIM host"
        cd src/main
        ./omnia.sh --run image_build_manager
        ```

    === "Using ansible-playbook"

        ```bash title="Run on: OIM host"
        source /opt/omnia/activate-omnia.sh
        cd src/image_build_manager/playbooks
        ansible-playbook image_build_manager.yml
        ```

    Tags are mutually exclusive. Run one tag at a time, or no tag for the
    default prepare-and-build flow.

## Verification

1. Verify the output contract:

    ```bash title="Run on: OIM host"
    cat /opt/omnia/image_build_manager/output/project_default/build_status.yml
    ```

    If `IMAGE_BUILD_MANAGER_DATA_PATH` or `OMNIA_PROJECT_NAME` is customized,
    use
    `<IMAGE_BUILD_MANAGER_DATA_PATH>/output/<OMNIA_PROJECT_NAME>/build_status.yml`.

    Confirm all of the following:

    - `overall_status` is `success`.
    - `image_build_type` records the engine used for this build.
    - `s3_configurations.endpoint_url` is the S3 HTTP or HTTPS endpoint and
      `s3_configurations.bucket` is `boot-images`.
    - Every expected architecture and functional group appears under
      `functional_group_images`.
    - Every functional-group entry has non-empty `kernel`, `initrd`, and
      `image` values. These values are exact endpoint-relative object paths;
      they include the bucket name, omit `s3://` and the endpoint, and end in
      an artifact filename.

    A successful `image-thrillhouse` entry has this form:

    ```yaml title="Expected structure"
    overall_status: "success"
    image_build_type: "image-thrillhouse"
    s3_configurations:
      endpoint_url: "http://10.20.0.1:9000"
      bucket: "boot-images"
    functional_group_images:
      - x86_64:
          - functional_group: "slurm_node_x86_64"
            kernel: "boot-images/slurm_node_x86_64/<image-name>-imgth/10.0/vmlinuz"
            initrd: "boot-images/slurm_node_x86_64/<image-name>-imgth/10.0/initramfs.img"
            image: "boot-images/slurm_node_x86_64/<image-name>-imgth/10.0/rootfs.squashfs"
    ```

    For `image-builder`, the kernel and initrd paths use the
    `boot-images/efi-images/<functional_group>/<image-name>-imgbld/` prefix and
    versioned filenames. The root filesystem is stored below
    `boot-images/<functional_group>/<image-name>-imgbld/`.

    Build an artifact URL by joining the values exactly once:

    ```text
    <s3_configurations.endpoint_url>/<kernel|initrd|image path>
    ```

    The workflow also writes a versioned copy named
    `build_status_<OMNIA_VERSION>_<YYYYMMDD_HHMM>.yml` beside the latest file.

2. Verify the build artifacts and services:

    ```bash title="Run on: OIM host"
    s3cmd ls -r s3://boot-images
    systemctl status registry.service
    ```

    When using MinIO, also verify:

    ```bash title="Run on: OIM host"
    systemctl status minio.service
    s3cmd ls
    ```

    The bucket listing must include `s3://boot-images` and `s3://efi`.

3. If verification fails, inspect the logs:

    - Main playbook log: `/var/log/omnia/image_build_manager/image_build_manager.log`
    - Runtime and per-image logs:
      `<IMAGE_BUILD_MANAGER_DATA_PATH>/log/<OMNIA_PROJECT_NAME>/`

## Next steps

- Continue to the [provisioning workflow](../orchestrator/provision_nodes.md).
  It consumes `build_status.yml` to validate images and create the
  `boot-service` configurations used during PXE boot.
- Retain `build_status.yml` and its S3 artifacts while they are needed for
  provisioning. The full cleanup flow removes the services, credentials,
  build output, buckets, and artifacts.
- When package inputs change, run the build again. Set `force_rebuild: true`
  when the package-hash cache must be bypassed; set `backup_s3_images: true` to
  preserve the existing compute artifacts under `*_prev` before rebuilding.

## Troubleshooting

- **A required environment variable is reported as missing**: When using
  `omnia.sh`, confirm that OIM setup installed the correct values in
  `/etc/omnia/omnia.env`. When running `ansible-playbook` directly, export
  `SYSTEM_ADMIN_NIC_IPV4`, `SYSTEM_HOSTNAME`, `SYSTEM_DOMAIN_NAME`,
  `OMNIA_DATA_PATH`, and `OMNIA_VERSION` in the same shell. Run the `precheck`
  tag again. If the administrative IP check fails, confirm that the address is
  assigned to a local OIM interface.

- **`repo_status.yml` is missing or rejected**: Confirm that
  `repo_manager_output_path` points to the Repo Manager output for the current
  project. The file must report `overall_status: "success"`, contain at least
  one usable x86_64 or aarch64 repository URL, and reference an existing certificate when
  `repo_manager.certificates.server_crt` is set. The build also fails when a
  listed repository URL is unreachable.

- **Catalog validation fails**: Follow
  [Select or update the catalog](../main/update_catalog.md) and confirm that
  `CATALOG_FILE_PATH` resolves to an existing JSON file. Confirm that it contains
  `functionallayer`, `groups`, and `packages` data and that its layer names end
  in the architecture being built. Names beginning with `baseos` are treated
  as base layers; all other matching layers are treated as compute layers.

- **No functional groups are found in config mode**: Confirm that
  `package_groups.yml` contains at least one key under `functional_groups` with
  the `_x86_64` or `_aarch64` suffix for the target architecture. Functional
  groups are not read from `image_build_config.yml`.

- **A package cannot be resolved**: Use the RPM package name rather than a
  command or binary name. Confirm that the package is available through one of
  the repository URLs in `repo_status.yml`, and synchronize the missing
  package in the Repo Manager when necessary.

- **MinIO or registry preparation fails**: Check
  `systemctl status minio.service` or `systemctl status registry.service`. The
  local endpoints use ports 9000 and 9001 for MinIO and port 5000 for the OCI
  registry. The registry uses HTTP; if `regctl` attempts HTTPS, configure it
  with:

    ```bash title="Run on: OIM host"
    /usr/local/bin/regctl registry set --tls disabled <SYSTEM_ADMIN_NIC_IPV4>:5000
    ```

- **The PowerScale endpoint is rejected**: Confirm that
  `s3_configurations.provider` is `powerscale`, `endpoint_url` is a reachable
  HTTP or HTTPS URL, and the Vault-encrypted credentials contain a non-empty
  `s3_access_id` and `s3_secret_key`.

- **An aarch64 build fails before image creation**: Confirm that the configured
  host responds to ping, port 22 is open, `uname -m` returns `aarch64`, and SSH
  credentials are correct. If the builder image or `regctl` cannot be obtained,
  make the OIM Repo Manager reachable from the ARM host or provide internet
  access for the fallback download.

- **A build fails or times out**: Review the per-image log in
  `<IMAGE_BUILD_MANAGER_DATA_PATH>/log/<OMNIA_PROJECT_NAME>/`. Increase
  `build_image.build_timeout` within its supported range or reduce
  `build_image.max_parallel` when the OIM does not have enough resources for
  concurrent builds.

- **Repository TLS verification fails**: Correct the certificate path in
  `repo_status.yml` and install a valid certificate. For repositories that use
  self-signed certificates, the input contract also supports setting
  `build_image.repo_ssl_verify: false`.
