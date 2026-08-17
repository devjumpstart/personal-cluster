# Mealie Kubernetes Deployment

This directory contains the reusable base Kubernetes manifests for deploying [Mealie](https://mealie.io/), a self-hosted recipe manager and meal-planning application.

The base defines the core application configuration shared across environments. Environment-specific settings—such as persistent storage, service exposure, ingress, and resource tuning—belong in Kustomize overlays.

## Base Resources

The base currently defines:

- A dedicated Kubernetes namespace
- The Mealie application deployment

Keeping the base minimal makes it portable, predictable, and easy to extend for different environments.

## Directory Structure

```text
mealie/
├── deployment.yaml
├── namespace.yaml
├── kustomization.yaml
└── README.md
```

## Deployment Details

| Setting | Value |
| --- | --- |
| Container image | `ghcr.io/mealie-recipes/mealie:v3.22.0` |
| Container port | `9000` |
| Namespace | `mealie` |

## Kustomize Design

This directory is the reusable application base. Environment overlays reference it and add or modify configuration without duplicating the core manifests.

```text
apps/
├── base/
│   └── mealie/
│       ├── deployment.yaml
│       ├── namespace.yaml
│       ├── kustomization.yaml
│       └── README.md
└── staging/
    └── mealie/
        ├── deployment-patch.yaml
        ├── pvc.yaml
        ├── service.yaml
        └── kustomization.yaml
```

For example, a staging overlay can extend the base with:

- Persistent storage
- Service exposure
- Environment-specific deployment patches
- Ingress and resource configuration

This separation keeps common application configuration in one place while allowing each environment to declare only its differences.

## GitOps Workflow

Mealie is managed declaratively through Git. Changes should follow this workflow:

1. Update the Kubernetes manifests.
2. Validate the rendered configuration locally:

   ```bash
   kubectl kustomize apps/staging/mealie
   ```

3. Commit and push the changes to the Git repository.
4. Allow Flux to reconcile the desired state into the cluster.
5. Verify the reconciliation and deployed resources:

   ```bash
   flux get kustomizations
   kubectl get all -n mealie
   ```

Git is the source of truth for this deployment. Direct changes to cluster resources should be avoided because Flux may revert them during reconciliation.
