# Prepare Worker-to-BMC Network Access

## Overview

When iDRAC Telemetry synchronizes its BMC inventory, it first delegates Redfish
validation and telemetry enablement to the first service Kubernetes worker in
the configured inventory. If that worker cannot be reached over SSH, it tries
the second service worker when one exists, then falls back to the Kubernetes
VIP. Each BMC must be reachable over HTTPS from the selected host.

The Telemetry source does not create VLAN interfaces or routes. Those settings
must already provide the connectivity required by the iDRAC workflow.

## Prerequisites

- Enable iDRAC Telemetry and provide a valid BMC CSV.
- Ensure `cluster_inventory` contains
  `service_kube_node_x86_64.hosts` entries with `ansible_host` values.
- Provide common BMC credentials through the Telemetry credential workflow.
- Enable the Redfish API on each BMC.

## Procedure

1. Identify the first service worker in the inventory referenced by
   `telemetry_config.yml`.

2. From the Kubernetes VIP, confirm that the worker accepts the same SSH check
   used by Telemetry:

    ```bash title="Run on: Kubernetes VIP"
    ssh -o ConnectTimeout=10 -o BatchMode=yes -o StrictHostKeyChecking=no \
      <worker-address> echo reachable
    ```

3. Ensure the worker can connect to every BMC address from the configured CSV
   at `https://<BMC_IP>/redfish/v1/`. Telemetry uses HTTP basic authentication,
   a 30-second timeout, and accepts the BMC's self-signed certificate for this
   check.

4. When site VLANs or routes are required, configure them through the site's
   network management process before deploying Telemetry. No VLAN variables or
   VLAN configuration playbook exist in the Telemetry source.

5. Deploy iDRAC Telemetry:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags deploy
    ```

## Verification

Review `<OMNIA_DATA_PATH>/telemetry/idrac_telemetry_report.yml`. BMCs that pass
reachability, authentication, Redfish, firmware, and license checks are listed
as enabled; failures are separated into invalid, unreachable, Redfish-disabled,
or unsupported results.

## Next steps

- Complete [Configure iDRAC Telemetry](configure_idrac.md).

## Troubleshooting

- **The worker SSH check fails:** Restore SSH reachability from the VIP. The
  workflow retries the second service worker, when present, and then falls back
  to the VIP for BMC validation, but worker access is the intended path.
- **A BMC returns `401`:** Correct the common BMC credentials.
- **A BMC returns `404`:** Enable its Redfish API.
- **A BMC times out or has a connection error:** Correct the external network
  path. Telemetry reports the BMC as unreachable and does not configure the
  missing VLAN or route.
