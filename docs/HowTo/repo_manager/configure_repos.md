# Create Local Repositories

## Overview

Repo Manager creates the local HTTPS Pulp service used by Omnia image-building
and provisioning workflows. It reads three customer inputs:

| Input | Purpose |
|---|---|
| Catalog JSON from `CATALOG_FILE_PATH` | Selects functional layers, groups, packages, OS versions, architectures, and sources |
| [`repo_manager_config.yml`](../../Reference/Configuration/repo_manager_config.md) | Maps catalog RPM sources and private registries to reachable upstream endpoints |
| [`repo_manager_endpoint_config.yml`](../../Reference/Configuration/repo_manager_endpoint_config.md) | Sets the host-facing Pulp IP and HTTPS port |

Repo Manager processes catalog contexts in ascending OS minor-version order.
For each context, a catalog RPM source is matched by `version`, `architecture`,
and `reponame`; an image source is matched by `registry`.

## Prerequisites

- Complete the prerequisites on the [Repository Manager](index.md) page.
- [Select or update the catalog](../main/update_catalog.md), and set
  `CATALOG_FILE_PATH` to the selected JSON file. Each functional layer must
  reference exactly one group with `type: "base_os"`, and every group and
  package reference must resolve.
- Ensure all selected source URLs are reachable from the OIM.
- [CRI-O repository URL is unreachable from OIM](../../Troubleshooting/repo_manager/repo_manager.md#cri-o-repository-url-is-unreachable-from-oim).
- [EPEL Repository Unavailable/Unstable/Too Slow](../../Troubleshooting/repo_manager/repo_manager.md#epel-repository-unavailableunstabletoo-slow).
- Have credentials available for the Pulp administrator and for any private
  registries that use basic authentication. Docker Hub credentials are
  optional for anonymous public pulls.
- If custom Pulp storage paths are configured in the source variables, create
  each directory and make it writable before running `prepare`; Repo Manager
  does not create filesystems or mount storage.

## Procedure

### 1. Load the environment

```bash title="Run on: OIM host"
cd <OMNIA_SOURCE_PATH>
./src/main/omnia.sh --setup-venv
source /opt/omnia/venv/bin/activate
set -a
source /etc/omnia/omnia.env
set +a
```

At minimum, the environment must contain:

```bash
SYSTEM_ADMIN_NIC_IPV4=<OIM-admin-network-IPv4>
CATALOG_FILE_PATH=/absolute/path/to/catalog_rhel.json
```

For catalog choices and the persistent environment configuration, follow
[Select or update the catalog](../main/update_catalog.md).

`OMNIA_DATA_PATH` defaults to `/opt/omnia`, and `OMNIA_PROJECT_NAME` defaults
to `project_default`. `REPO_MANAGER_DATA_PATH` can override the Repo Manager
runtime root for playbook execution.

### 2. Configure RPM repositories

Edit the staged runtime input
`$OMNIA_DATA_PATH/repo_manager/input/$OMNIA_PROJECT_NAME/repo_manager_config.yml`
(default `/opt/omnia/repo_manager/input/project_default/repo_manager_config.yml`).
If the file does not exist, run `domain-init.sh` to stage the source templates
from `src/repo_manager/input/`. The minimum structure is:

```yaml
repo_config: partial
caching_policy: true

registries:

repositories:
  "10.0":
    x86_64:
      baseos: {}
      appstream: {}
      epel:
        url: "https://mirror.example/rhel/10/epel/x86_64/"
        gpgkey: "https://mirror.example/keys/RPM-GPG-KEY-EPEL-10"
        policy: partial
        caching: true
        priority: 99
```

Use the catalog source values to build the lookup path. For example, this
source:

```json
{
  "architecture": "x86_64",
  "name": "rhel",
  "version": ["10.0"],
  "reponame": "epel"
}
```

requires `repositories."10.0".x86_64.epel`.

The exact keys `baseos`, `appstream`, and `codeready-builder` may be empty when
the OIM has usable subscription content. Repo Manager prefers the matching EUS
repository and falls back to the standard subscription repository. An explicit
URL always takes precedence. Without usable subscription access, every
catalog-referenced repository requires a non-empty URL.

Repository entries accept `url`, `gpgkey`, `policy`, `caching`, `priority`,
`sslcacert`, `sslclientkey`, and `sslclientcert`. `priority` must be from 1
through 100.

### 3. Choose the RPM content policy

Repo Manager combines two independent settings to decide what Pulp mirrors and
what `dnf` does on the OIM:

- `repo_config` and `caching_policy` are the global defaults that apply to
  every repository.
- A repository entry under `repositories.<version>.<arch>.<repo_name>` can
  override them with `policy` and `caching`.

Each combination maps to a Pulp download policy (`immediate`, `on_demand`, or
`streamed`) that controls what Pulp synchronizes. `dnf` always operates on the
catalog-selected packages; it either downloads them with dependencies into the
Repo Manager RPM directory or validates them against Pulp, based on the
resolved Pulp policy.

#### Global `repo_config` + `caching_policy`

| `repo_config` | `caching_policy` | Pulp policy | What Repo Manager does per catalog-selected package | Use when |
|---|---|---|---|---|
| `always` | `false` | `immediate` | Runs `dnf download <pkg>` and its dependencies from the local Pulp copy into the Repo Manager RPM directory. | Air-gapped or fully offline deployments requiring a complete local mirror. |
| `partial` | `true` | `on_demand` | Runs `dnf download <pkg>` and its dependencies; Pulp fetches and retains the RPMs. | Selective sync with retention. |
| `partial` | `false` | `streamed` | Runs `dnf info <pkg>` to validate metadata against the Pulp mirror; no RPMs are downloaded. | Validation only; upstream must remain reachable at install time. |

#### Per-repository `policy` + `caching`

| `policy` | `caching` | Pulp policy | What Repo Manager does per catalog-selected package | Use when |
|---|---|---|---|---|
| `always` | `false` | `immediate` | Runs `dnf download <pkg>` and its dependencies from local Pulp. | This repository must be fully self-contained offline. |
| `partial` | `true` | `on_demand` | Runs `dnf download <pkg>` and its dependencies; Pulp fetches and retains the RPMs. | Only catalog-selected packages from this repository are needed. |
| `partial` | `false` | `streamed` | Runs `dnf info <pkg>` to validate metadata; no RPMs are downloaded. | Selected packages are validated only; upstream reachable at install time. |

A catalog item with `packagetype: "rpm_repo"` requires retained content and
must not resolve to `streamed`. Container synchronization uses an independent
policy and defaults to `immediate`.

### 4. Configure private registries when required

Known public registries can be used without a `registries` entry. For a private
registry, add a mapping such as:

```yaml
registries:
  private_registry:
    base_url: "https://<registry_host>"
    port: 443
    auth:
      type: basic
      credentials:
        vault_path: "registries/<registry_key>"
    tls:
      ca_path: "/path/to/<registry_ca>.crt"
      client_cert_path: ""
      client_key_path: ""
      insecure: false

  insecure_registry:
    base_url: "http://<registry_host>"
    port: <registry_port>
    auth:
      type: none
    tls:
      insecure: true
```

Replace `<registry_host>` with the registry IP address or FQDN, `<registry_port>`
with the actual port, and `<registry_ca>.crt` with the path to the CA
certificate. `auth.type` is `none` or `basic`; basic authentication requires a
`vault_path` that Repo Manager collects during `prepare` and stores in an
Ansible Vault file. Do not put credentials in the catalog or repository
configuration.

The image's catalog source must use `registry: "private_registry"`, while its
package `name` must start with the actual configured `host[:port]`, for example
`<registry_host>:<registry_port>/library/image`.

### 5. Configure the Pulp endpoint

Edit the staged runtime input
`$OMNIA_DATA_PATH/repo_manager/input/$OMNIA_PROJECT_NAME/repo_manager_endpoint_config.yml`
(default `/opt/omnia/repo_manager/input/project_default/repo_manager_endpoint_config.yml`).
If the file does not exist, run `domain-init.sh` to stage the source templates
from `src/repo_manager/input/`.

```yaml
pulp_server_port: 2225
# Optional; SYSTEM_ADMIN_NIC_IPV4 is used when omitted.
# pulp_server_ip: "192.0.2.10"
```

The selected host port maps to port `443` in the Pulp container. HTTPS is
mandatory, and certificate paths are derived automatically.

### 6. Validate the inputs

```bash title="Run on: OIM host"
cd <OMNIA_SOURCE_PATH>/src/repo_manager/playbooks
ansible-playbook repo_manager.yml --tags precheck
```

Run `precheck` against the staged runtime inputs, catalog, and subscription
mappings. If the runtime inputs are missing, run `./domain-init.sh` from
`src/repo_manager/` first to stage the source templates.

### 7. Deploy Pulp, synchronize content, and generate status

```bash title="Run on: OIM host"
ansible-playbook repo_manager.yml \
  --tags "prepare,precheck,download,status"
```

Credentials are stored in
`<REPO_MANAGER_DATA_PATH>/input/<project>/repo_manager_config_credentials.yml`
with the matching `.repo_manager_config_credentials_key`. Both files are
root-owned, mode `0600`, and the credential YAML is encrypted with Ansible
Vault.

## Verification

### 1. Verify the output contract for image building

Repo Manager publishes the synchronized repository information for Image Build
Manager and cluster provisioning workflows at:

```text
<REPO_MANAGER_DATA_PATH>/output/<project>/repo_status.yml
```

The default path is
`/opt/omnia/repo_manager/output/project_default/repo_status.yml`. Inspect the
file before starting an image build:

```bash title="Run on: OIM host"
cat /opt/omnia/repo_manager/output/project_default/repo_status.yml
```

The generated contract has this structure; versions, architectures,
repository names, and URLs reflect the active catalog and the distributions
available in Pulp:

```yaml
overall_status: "success"
cluster_os_type: "rhel"
repo_config: "partial"
execution_contexts:
  - context_id: "rhel_10.0"
    os_type: "rhel"
    os_version: "10.0"
    architectures: ["x86_64"]
overall_status_by_version:
  "10.0": "success"
repo_manager:
  port: 2225
  certificates:
    server_crt: "/opt/omnia/repo_manager/pulp_config/settings/certs/pulp_webserver.crt"
    certs_dir: "/opt/omnia/repo_manager/pulp_config/settings/certs"
repositories:
  "10.0":
    x86_64:
      baseos:
        url: "https://192.0.2.10:2225/pulp/content/.../baseos/"
```

Before proceeding to Image Build Manager, verify the following contract
conditions:

| Check | Expected value |
|---|---|
| Aggregate readiness | `overall_status` is `success` |
| Selected OS versions | Every entry in `overall_status_by_version` is `success` |
| Catalog context | `execution_contexts` contains the required OS version and architecture |
| RPM content | `repositories.<version>.<architecture>` contains every catalog-required repository and its Pulp `url` |
| HTTPS access | `repo_manager.port`, `repo_manager.certificates.server_crt`, and `repo_manager.certificates.certs_dir` identify the Pulp endpoint and trust certificate |

Repository entries can also contain `priority` when it was explicitly
configured. Depending on the selected catalog, the contract can contain
`file_repos`, non-secret `registries` settings, content-type base URLs, and
backward-compatible `offline_*_path` values.

Image Build Manager must trust the Pulp CA certificate and must not consume a
contract whose `overall_status` is not `success`. If a catalog-required RPM
distribution is missing, Repo Manager writes `overall_status: failed`, marks
the affected version as `failed`, and leaves the corresponding repository maps
without consumable URLs. Correct the synchronization failure and rerun the
`download,status` tags before building the image.

### 2. Verify Pulp and synchronized content

Verify the service and inspect each Pulp content family used by the catalog:

```bash title="Run on: OIM host"
systemctl status pulp.service
pulp status
pulp rpm distribution list --limit 1000
pulp container distribution list --limit 1000
pulp file distribution list --limit 1000
pulp python distribution list --limit 1000
```

## Next steps

- [Build Cluster Images](../image_build_manager/build_images.md).
- [Select or update the catalog](../main/update_catalog.md) before rerunning
  Repo Manager for a different workload, architecture, or VAST option.
- [Configure and add packages to the catalog](adding_additional_packages.md).
- [Configure a new RPM repository](adding_additional_repositories.md).
- [Update synchronized content after catalog changes](../../Operations/repo_manager/updating_local_repositories.md).

## Troubleshooting

- **`Additional properties are not allowed`**: Remove unknown keys and use the
  implemented version → architecture → repository structure. Configuration
  keys are lowercase.
- **A referenced RPM repository is missing**: Add the exact catalog
  `reponame` under the matching version and architecture. A mapping for one
  architecture does not satisfy another.
- **An empty subscription repository fails**: Check the subscription with
  `subscription-manager identity`, `subscription-manager status`, and
  `subscription-manager repos --list-enabled`. If subscription content is
  unavailable, provide an explicit URL.
- **Private registry validation fails**: Check the complete chain from catalog
  `source.registry`, through `registries.<key>` and `vault_path`, to the
  encrypted registry credential entry. Rerun `prepare` to collect credentials.
- **Pulp health validation fails**: Inspect `systemctl status pulp.service` and
  `podman logs --tail 200 pulp`, correct storage or registry access, and rerun
  `prepare`.
- **Synchronization fails for one package**: Inspect the package status and
  worker logs below
  `<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/<architecture>/` and rerun
  `download` after fixing the source.
