# OIM Requirements

This section outlines the key requirements for the Omnia Infrastructure Manager
(OIM) used to deploy HPC clusters. For more information about the supported
devices and software, see [Support Matrix](../index.md#support-matrix).

## Omnia Infrastructure Manager

- Choose a **server outside of your intended cluster** that meets the required [Storage Requirements](disk_space.md) to function as the Omnia Infrastructure Manager (OIM).
- Ensure the OIM has at least 64 GB RAM. To check the free RAM size, use the `free -h` command. To check the disk space, use the `df -h` command.
- Ensure that the OIM has the RHEL operating system installed with the **Server with GUI** Base Environment. For a complete list of supported RHEL versions, see the [supported operating systems](../SupportMatrix/operating_systems.md).
- Ensure that **Podman** container engine is installed on the OIM.
- The OIM must have access to the admin (PXE) network and to every configured
  package, container, and source repository. Internet access is required when
  public repositories are used; an offline deployment can use reachable local
  mirrors instead.
- Verify that **Git** is installed. If not, install it using:

    ```bash title="Run on: OIM host"
    dnf install git -y
    ```

- All target bare-metal servers (cluster nodes) must be **reachable from the OIM**.
- Make sure that the required ports are open on the OIM node for cluster deployment. For detailed information on the required ports, refer to [Ports Used by the OIM](../../GetStarted/prerequisites_checklist.md#ports-used-by-the-oim).
- Complete [Setup the OIM](../../HowTo/main/setup_oim.md) to install the shared
  environment and initialize the deployment modules.

!!! info

    See [Running Deployment Modules](../../Overview/domain_execution.md) for
    the supported module setup and execution workflow.
















