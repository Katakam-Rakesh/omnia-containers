# repo_manager_config.yml

This file defines Repo Manager synchronization policy, RPM repository mappings,
and optional container registry endpoints.

## Location

```text
$OMNIA_DATA_PATH/repo_manager/input/$OMNIA_PROJECT_NAME/repo_manager_config.yml
```

The default location is
`/opt/omnia/repo_manager/input/project_default/repo_manager_config.yml`.

## Top-level parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `repo_config` | string | Yes | Global RPM policy: `always` or `partial`. |
| `caching_policy` | boolean | No | Global Pulp caching behavior. The source value is `true`. |
| `repositories` | object | Yes | Repository definitions organized by OS version and architecture. |
| `registries` | object or null | No | Container registries keyed by registry name. |

Unknown top-level and nested properties are rejected.

## Repository entries

Each version can contain `x86_64` and `aarch64` mappings. A repository entry
supports the following fields:

| Field | Type | Values or constraint |
|---|---|---|
| `url` | string or null | Repository URL; may be empty for subscription-provided RHEL repositories. |
| `gpgkey` | string or null | GPG key URL. |
| `policy` | string | `always`, `partial`, or `never`. |
| `caching` | boolean | Per-repository caching override. |
| `priority` | integer | DNF priority from 1 through 100. |
| `sslcacert` | string or null | CA certificate path. |
| `sslclientkey` | string or null | mTLS client-key path. |
| `sslclientcert` | string or null | mTLS client-certificate path. |

`user_repos` contains named repository entries using the same fields.

## Registry entries

A registry requires `base_url`, `port`, and `auth`. `auth.type` is `none` or
`basic`. Basic authentication also requires
`auth.credentials.vault_path`, which selects an entry from the encrypted Repo
Manager credential file. The optional `tls` mapping supports `ca_path`,
`client_cert_path`, `client_key_path`, and `insecure`.

A catalog image source that uses the `registry` field must have a `name` that
starts with the configured registry authority, i.e. `<host>[:<port>]/<image_path>`.

## Usage example

```yaml title="File: /opt/omnia/repo_manager/input/project_default/repo_manager_config.yml"
repo_config: "partial"
caching_policy: true

registries:

repositories:
  "10.0":
    x86_64:
      baseos: {}
      appstream: {}
      codeready-builder: {}
      epel:
        url: "https://dl.fedoraproject.org/pub/epel/10/Everything/x86_64/"
        gpgkey: "https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-10"
        policy: "partial"
        caching: true
        priority: 99
```

Credentials are collected by the Repo Manager credential workflow and stored
in `repo_manager_config_credentials.yml` with the matching
`.repo_manager_config_credentials_key`; do not place passwords in this file.

## Related configuration

- [Repo Manager endpoint](repo_manager_endpoint_config.md)
- [Repo Manager contract](../domain_contracts/repo_manager_contract.md)
