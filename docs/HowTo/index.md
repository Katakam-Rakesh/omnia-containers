# How-to Guides


Task-oriented procedures for deploying, configuring, and managing Omnia
clusters using its modular deployment architecture. Each guide follows a consistent structure: **Overview**, **Prerequisites**,
**Procedure**, **Verification**, **Next Steps**, and **Troubleshooting**.

!!! tip

    If you are new to Omnia, start with the [Get Started](../GetStarted/index.md) tutorials
    first. How-to guides assume you understand Omnia's architecture and have a
    working OIM.

## Shared controller

Main prepares the common environment and coordinates module execution. It is
not a deployment module.

| Area | Description | Index page |
|---|---|---|
| **main** | Setup, initialization, and cross-module coordination | [Main](main/index.md) |

## Deployment module guides

Omnia organizes deployment capabilities into seven modules. Each module has
its own task-oriented guides:

| Deployment module | Description | Index page |
|---|---|---|
| **repo_manager** | Repository mirroring and package synchronization | [Repository Manager](repo_manager/index.md) |
| **image_build_manager** | OS image building and S3 storage | [Image Build Manager](image_build_manager/index.md) |
| **discovery** | Node inventory and PXE mapping file generation | [Discovery](discovery/index.md) |
| **orchestrator** | Slurm, Kubernetes, networking, storage, authentication | [Orchestrator](orchestrator/index.md) |
| **telemetry** | iDRAC, LDMS, storage, and fabric metrics collection | [Telemetry](Telemetry/index.md) |
| **build_stream** | GitLab CI/CD pipeline automation | [BuildStreaM](build_stream/index.md) |
| **utils** | Helper utilities for log backup, Slurm configuration management, and OS installation | [Utilities](utils/index.md) |

## Module-specific procedures

Select a module above to view its how-to guides. Depending on the module, these include:

- Configuration procedures
- Deployment workflows
- Verification steps
- Troubleshooting guidance

!!! note

    Modules can be invoked separately, but downstream modules require the
    contracts produced upstream. A typical direct deployment follows:
    Repository Manager → Image Build Manager → optional Discovery →
    Orchestrator → optional Telemetry. BuildStreaM provides a separate GitLab
    CI/CD automation path; Utilities runs on demand.

















