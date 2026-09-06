# Audiobookshelf on K3s

A production-minded, GitOps-managed deployment of [Audiobookshelf](https://www.audiobookshelf.org/) on K3s.

This project separates application state from user-owned media, uses NFS to make the media library available across cluster nodes, and publishes the application through Cloudflare Tunnel without exposing a Kubernetes node directly to the internet.

## Overview

The deployment combines:

- K3s
- Flux CD
- Kustomize
- Rancher Local Path Provisioner
- NFS-backed media storage
- Cloudflare Tunnel and Cloudflare Access
- SOPS with age encryption
- Renovate dependency updates

The central storage principle is simple:

| Container path | Purpose | Storage |
|---|---|---|
| `/config` | Audiobookshelf application state | Kubernetes PVC |
| `/metadata` | Application-managed metadata | Kubernetes PVC |
| `/audiobooks` | User-owned audiobook library | Read-only NFS mount |
| `/podcasts` | Podcast library and downloads | Read-write NFS mount |

Application state follows the Kubernetes workload. Media remains outside Kubernetes and can survive workload or cluster recreation independently.

## Architecture

```text
                           Internet
                              |
                       Cloudflare Access
                              |
                       Cloudflare Tunnel
                              |
                     cloudflared in K3s
                              |
                   ClusterIP Service :80
                              |
                     Audiobookshelf Pod
                    /         |         \
                   /          |          \
              /config     /metadata      Media
                 |            |          /   \
                PVC          PVC        /     \
                 |            |   /audiobooks /podcasts
             local-path   local-path     |         |
                                       NFS ro    NFS rw
                                          \       /
                                           NFS server
                                               |
                                      Dedicated filesystem
```

The `cloudflared` workload runs inside the cluster and connects directly to the internal ClusterIP Service. A `NodePort` or public Kubernetes load balancer is not required.

## Design decisions

### Kubernetes PVCs for application state

`/config` and `/metadata` contain writable state created by Audiobookshelf. Despite its name, `/config` is not a Kubernetes ConfigMap. Both paths require persistent writable filesystems.

The deployment uses the K3s `local-path` StorageClass with `ReadWriteOnce` claims. This is a practical choice for a single-replica homelab workload, but it has an important limitation: the data is stored on one cluster node and is not automatically replicated to other nodes.

### NFS for media

Audiobooks and podcasts are user-owned data rather than application state. Hosting them on an external NFS filesystem provides:

- independence from PVC and cluster lifecycles;
- access from any appropriately configured cluster node;
- ordinary files that can be managed and backed up outside Kubernetes;
- separate read-only and read-write policies for different libraries.

Audiobooks are exported read-only to reduce the risk of accidental modification. Podcasts are read-write so Audiobookshelf can manage downloaded episodes.

### One application replica

The deployment runs one Audiobookshelf replica. This matches the `ReadWriteOnce` state volumes and avoids concurrent writers to application state. The design prioritizes simplicity and recoverability rather than high availability.

## Prerequisites

- A working K3s cluster
- Flux installed and connected to a Git repository
- Kustomize or `kubectl` with Kustomize support
- A reachable NFS server
- NFS client packages installed on every schedulable K3s node
- A Cloudflare account and an existing Cloudflare Tunnel
- SOPS and age for encrypted configuration
- A DNS zone managed by Cloudflare

## Configuration

Replace all example values before deployment:

| Placeholder | Description |
|---|---|
| `<NFS_SERVER_IP>` | Address of the NFS server reachable from every cluster node |
| `<AUDIOBOOKS_EXPORT>` | Server export containing audiobooks |
| `<PODCASTS_EXPORT>` | Server export containing podcasts |
| `<AUDIOBOOKSHELF_VERSION>` | Verified container image tag |
| `audiobooks.example.com` | Public hostname protected by Cloudflare Access |
| `<TUNNEL_NAME>` | Existing Cloudflare Tunnel name or ID |

Do not commit credentials, private keys, tunnel tokens, internal inventory, or decrypted SOPS files.

## Kubernetes resources

### PersistentVolumeClaims

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-config
  namespace: audiobookshelf
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-metadata
  namespace: audiobookshelf
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 512Mi
```

Confirm that the selected StorageClass supports the required binding and expansion behavior. Increasing a requested size does not guarantee that an existing volume can be expanded.

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
  labels:
    app: audiobookshelf
spec:
  replicas: 1
  selector:
    matchLabels:
      app: audiobookshelf
  template:
    metadata:
      labels:
        app: audiobookshelf
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: audiobookshelf
          image: ghcr.io/advplyr/audiobookshelf:<AUDIOBOOKSHELF_VERSION>
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          env:
            - name: TZ
              value: UTC
          volumeMounts:
            - name: config
              mountPath: /config
            - name: metadata
              mountPath: /metadata
            - name: audiobooks
              mountPath: /audiobooks
              readOnly: true
            - name: podcasts
              mountPath: /podcasts
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: audiobookshelf-config
        - name: metadata
          persistentVolumeClaim:
            claimName: audiobookshelf-metadata
        - name: audiobooks
          nfs:
            server: <NFS_SERVER_IP>
            path: <AUDIOBOOKS_EXPORT>
            readOnly: true
        - name: podcasts
          nfs:
            server: <NFS_SERVER_IP>
            path: <PODCASTS_EXPORT>
            readOnly: false
```

Use an image tag that exists in the container registry. A GitHub release prefixed with `v` does not necessarily imply that the container image uses the same prefix.

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: audiobookshelf-svc
  namespace: audiobookshelf
  labels:
    app: audiobookshelf
spec:
  type: ClusterIP
  selector:
    app: audiobookshelf
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

## NFS exports

An example server configuration is:

```exports
<AUDIOBOOKS_EXPORT> <CLUSTER_NETWORK>(ro,sync,no_subtree_check)
<PODCASTS_EXPORT>   <CLUSTER_NETWORK>(rw,sync,no_subtree_check)
```

Apply and inspect the exports on the NFS server:

```bash
sudo exportfs -ra
sudo exportfs -v
```

Validate visibility from every eligible cluster node:

```bash
showmount -e <NFS_SERVER_IP>
```

The container runs as UID and GID `1000`. Ensure server-side numeric ownership and permissions allow that identity to read audiobooks and write podcasts. Retain the default `root_squash` behavior unless there is a well-understood reason to change it.

## Cloudflare Tunnel

Add an ingress rule to the existing `cloudflared` configuration:

```yaml
- hostname: audiobooks.example.com
  service: http://audiobookshelf-svc.audiobookshelf.svc.cluster.local:80
```

Create the DNS route:

```bash
cloudflared tunnel route dns <TUNNEL_NAME> audiobooks.example.com
```

Protect the hostname with a Cloudflare Access application and an explicit allow policy. Application authentication and Cloudflare Access are complementary security layers.

After changing a ConfigMap, restart `cloudflared` if the deployment does not reload configuration automatically.

## Secret management

Sensitive configuration should be encrypted with SOPS before it enters Git. The age private key must remain outside the repository and be backed up securely.

Recommended edit flow:

```bash
SOPS_AGE_KEY_FILE=/secure/path/to/age-key \
  sops decrypt --in-place path/to/config.enc.yaml

# Edit the file, then immediately re-encrypt it.

SOPS_AGE_KEY_FILE=/secure/path/to/age-key \
  sops encrypt --in-place path/to/config.enc.yaml
```

Before committing:

```bash
grep -n 'sops:' path/to/config.enc.yaml
git diff -- path/to/config.enc.yaml
git status
```

`SOPS_AGE_KEY_FILE` must contain a filesystem path. `SOPS_AGE_KEY` contains literal private-key text. Mixing them up can cause SOPS to interpret a private key as a filename.

## GitOps workflow

1. Create a feature branch from an up-to-date `main` branch.
2. Make the manifest and encrypted configuration changes.
3. Render the complete Kustomization locally.
4. Review the Git diff for sensitive or unintended content.
5. Commit and push the branch.
6. Open and review a pull request.
7. Merge the PR and allow Flux to reconcile the new revision.
8. Verify the rollout, storage mounts, Service, tunnel, and Access policy.

Render before committing:

```bash
kubectl kustomize path/to/audiobookshelf
```

Inspect Flux after merging:

```bash
flux get sources git
flux get kustomizations
```

Do not copy controller-generated Flux labels or resource `status` fields from live objects into source manifests.

## Renovate

Pin the Audiobookshelf container image to a verified version instead of using `latest`. Renovate can then propose version changes through pull requests, keeping upgrades visible and reversible in Git history.

Review release notes, confirm backup freshness, and validate the exact container tag before merging an update. Private container registries require registry credentials separate from source-control credentials; store those credentials in a secret rather than the Renovate configuration committed to Git.

## Validation

Check the workload:

```bash
kubectl get pods,pvc,svc,endpoints -n audiobookshelf
kubectl rollout status deployment/audiobookshelf -n audiobookshelf
kubectl logs -n audiobookshelf deployment/audiobookshelf
```

Check the runtime identity and mounts:

```bash
kubectl exec -n audiobookshelf deployment/audiobookshelf -- id
kubectl exec -n audiobookshelf deployment/audiobookshelf -- \
  ls -ld /config /metadata /audiobooks /podcasts
```

Expected behavior:

- `/config`, `/metadata`, and `/podcasts` are writable.
- `/audiobooks` is readable but not writable.
- Both PVCs are bound.
- The Service has a ready endpoint.
- The application responds through the in-cluster Service.
- Cloudflare Access challenges unauthenticated public requests.

## Backup and recovery

Back up each data class independently:

| Data | Backup requirement |
|---|---|
| `/config` | Application-consistent, versioned backup |
| `/metadata` | Versioned backup |
| Audiobooks | Separate copy on another device or system |
| Podcasts | Separate copy if the content must be retained |
| Git repository | Remote repository and normal source-control safeguards |
| age private key | Secure backup outside Git |

Git recreates declared resources; it does not restore PVC contents or media. Periodically test the full recovery process rather than relying only on successful backup-job reports.

## Troubleshooting

### Pod remains pending

Inspect Pod events and volume binding:

```bash
kubectl describe pod -n audiobookshelf -l app=audiobookshelf
kubectl get pvc,pv -n audiobookshelf
```

Local storage can constrain the Pod to the node holding its data.

### Image cannot be pulled

Confirm the registry path and exact tag. Release names and registry tags may differ.

### NFS mount fails

Verify the server is reachable, exports are loaded, NFS client packages exist on the scheduled node, the cluster network is permitted, and relevant firewall rules allow NFS.

### Permission denied

Compare the Pod's numeric UID/GID with server-side file ownership. `fsGroup` does not override an NFS read-only export or restrictive server permissions.

### Service has no endpoints

Compare the Service selector with the Pod labels and confirm the Pod is ready:

```bash
kubectl get svc,endpoints,pods -n audiobookshelf --show-labels
```

### Cloudflare cannot reach the application

Test from the inside out: Pod, Service endpoints, in-cluster DNS and HTTP, `cloudflared` logs, tunnel ingress rule, DNS route, and finally the Access policy.

## Security considerations

- Run the container as a non-root UID/GID.
- Use the runtime-default seccomp profile.
- Export audiobooks read-only and also mount them read-only in the Pod.
- Keep NFS restricted to the required trusted network or hosts.
- Preserve NFS `root_squash` behavior.
- Protect the public hostname with Cloudflare Access.
- Never commit decrypted secrets, tunnel credentials, private keys, or registry passwords.
- Review all automated dependency updates before merging.
- Keep K3s nodes, the NFS server, and container images patched.

## Operational limitations

This design is intentionally pragmatic rather than highly available:

- one application replica serves requests;
- application-state volumes are node-local;
- the NFS server is a storage dependency and possible single point of failure;
- the Cloudflare Tunnel is part of the external access path.

Monitor filesystem capacity, inode use, PVC usage, Pod restarts, NFS health, Flux reconciliation, tunnel health, and failed dependency updates.

## Further documentation

- [Audiobookshelf documentation](https://www.audiobookshelf.org/docs/)
- [K3s documentation](https://docs.k3s.io/)
- [Flux documentation](https://fluxcd.io/flux/)
- [Kustomize documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [SOPS documentation](https://getsops.io/)
- [Cloudflare Tunnel documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)

## License

No license is provided by this README. Add a `LICENSE` file before distributing or accepting contributions under specific terms.
