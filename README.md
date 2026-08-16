# Personal Cluster

## Overview

This repository contains the declarative configuration for my personal Kubernetes homelab: a single-node [k3s](https://k3s.io/) cluster running on my homelab server and managed through [Flux](https://fluxcd.io/).

The repository is the foundation for managing the cluster as code. It serves as a practical GitOps environment for developing repeatable deployment patterns, operating Kubernetes workloads, and progressively introducing self-hosted applications and supporting infrastructure.

Git is the source of truth for the cluster. Desired state is defined here, committed through version control, and reconciled by Flux rather than maintained through manual changes on the server.

## Architecture and Workflow

The repository follows a monorepo structure that keeps reusable application definitions, environment-specific configuration, and cluster bootstrap resources together.

```text
Git repository
      |
      | Flux reconciliation
      v
Single-node k3s cluster
      |
      +-- Cluster-level Kustomizations
              |
              +-- Staging application overlays
                      |
                      +-- Reusable base manifests
```

Flux continuously monitors this repository and reconciles the desired state into the cluster. Application manifests are composed with Kustomize, allowing shared definitions to remain separate from environment-specific configuration.

## Repository Structure

```text
.
├── apps
│   ├── base
│   │   └── linkding
│   └── staging
│       └── linkding
└── clusters
    └── staging
        ├── apps.yaml
        └── flux-system
            └── ...
```

### `apps/base/linkding`

An example of the application base pattern used in this repository. Base directories contain reusable Kubernetes resources that describe an application's common deployment requirements without coupling them to a specific cluster environment.

### `apps/staging/linkding`

An example of an environment overlay. Overlay directories compose resources from `apps/base` and apply configuration specific to the target environment, such as namespaces, patches, resource settings, or other deployment differences.

### `clusters/staging/apps.yaml`

Defines the cluster-level Flux `Kustomization` responsible for reconciling application configuration into the staging cluster. It connects the cluster to the desired application overlays and establishes the reconciliation boundary for workloads.

### `clusters/staging/flux-system/*`

Contains the Flux bootstrap resources generated when the cluster was connected to this repository. These files configure the Flux controllers and Git source required for continuous reconciliation. They form the foundation of the cluster's GitOps control plane and should generally be changed through the appropriate Flux bootstrap or upgrade workflow.

## GitOps Workflow

The standard change workflow is:

1. Define or update Kubernetes resources under `apps/base`.
2. Add environment-specific composition or patches under `apps/staging`.
3. Reference the desired application configuration from the staging cluster configuration.
4. Commit and push the change to Git.
5. Allow Flux to detect the new revision and reconcile the cluster toward the declared state.
6. Verify reconciliation and workload health in the cluster.

This approach provides a versioned history of cluster changes, makes configuration reviewable, and supports repeatable recovery from the repository's declared state.

## Goals and Next Steps

This repository is intended to grow alongside the cluster. Its direction includes:

- Adding self-hosted applications through a consistent base-and-overlay structure.
- Bringing supporting services and cluster infrastructure under declarative management.
- Developing reusable conventions for configuration, reconciliation, and environment separation.
- Improving observability, resilience, security, and operational documentation over time.
- Maintaining pull-based GitOps as the primary delivery and change-management model.

## Tech Stack

| Technology | Role |
|---|---|
| k3s | Lightweight Kubernetes distribution running the homelab cluster |
| Flux | GitOps controller that reconciles repository state into the cluster |
| Kustomize | Composition of reusable manifests and environment-specific overlays |
| Kubernetes | Workload orchestration and declarative resource management |
| Git | Version control and source of truth for cluster configuration |

## Project Status

This is an active personal infrastructure project and GitOps learning environment. The repository will continue to evolve as more workloads, platform services, and operational practices are brought under declarative management.
