# Cloudflare Tunnel (`cloudflared`)

This directory contains the Kubernetes configuration for running `cloudflared` in the staging homelab cluster.

The deployment creates an outbound-only Cloudflare Tunnel between the Kubernetes cluster and the Cloudflare edge. It provides secure public access to selected internal applications without exposing the cluster through a public `LoadBalancer`, opening inbound router ports, or publishing the home IP address as an application origin.

## Why this exists

The applications in this cluster use internal Kubernetes `ClusterIP` Services. Those Services are reachable inside the cluster, but they are not directly reachable from the public internet.

`cloudflared` bridges those two environments:

```text
Internet user
    ↓
Cloudflare DNS and edge
    ↓
Cloudflare Tunnel
    ↓ outbound connections from cloudflared
cloudflared pods in Kubernetes
    ↓
ClusterIP Service
    ↓
Application pod
```

This design was chosen to:

- expose selected homelab applications over HTTPS;
- avoid inbound firewall and port-forwarding rules;
- keep application Services private inside Kubernetes;
- use Cloudflare as the public edge and DNS provider;
- manage the in-cluster configuration through GitOps;
- keep tunnel credentials encrypted in Git with SOPS; and
- provide redundant tunnel connectors by running two replicas.

## What it exposes

The tunnel is named `homelab-staging` and currently routes these public hostnames:

| Public hostname | Kubernetes origin |
| --- | --- |
| `mealie.olakunle.ca` | `http://mealie-svc.mealie.svc.cluster.local:9000` |
| `linkding.olakunle.ca` | `http://linkding-svc.linkding.svc.cluster.local:9090` |

An unmatched hostname receives an HTTP `404` response from `cloudflared` rather than being sent to an unintended service.

## How routing works

Publishing an application requires two separate mappings.

### 1. Public DNS maps the hostname to the tunnel

Cloudflare DNS associates a public hostname with the `homelab-staging` tunnel. This makes requests for the hostname arrive at the correct tunnel on the Cloudflare edge.

For example:

```text
mealie.olakunle.ca
    ↓
homelab-staging tunnel
```

### 2. The cloudflared ingress configuration maps the hostname to a Service

Inside the cluster, `cloudflared` matches the request hostname and forwards the request to the corresponding Kubernetes Service:

```yaml
ingress:
  - hostname: mealie.olakunle.ca
    service: http://mealie-svc.mealie.svc.cluster.local:9000

  - hostname: linkding.olakunle.ca
    service: http://linkding-svc.linkding.svc.cluster.local:9090

  - service: http_status:404
```

The public DNS record and the ingress rule are both required. A missing DNS record results in a name-resolution failure; a missing or incorrect ingress rule prevents `cloudflared` from routing the request to the application.

## Deployment design

### Locally managed tunnel

This deployment uses a locally managed Cloudflare Tunnel. Its configuration and credentials are supplied to the `cloudflared` process as mounted files.

The tunnel identity is configured as:

```yaml
tunnel: homelab-staging
```

The credential contents are not stored in plaintext in the repository.

### Two replicas

The Deployment runs two `cloudflared` replicas. Each replica establishes outbound connections to the Cloudflare edge, providing connector redundancy if a pod is restarted or temporarily unavailable.

```yaml
replicas: 2
```

This improves availability at the connector layer. It does not make an origin application highly available if that application itself has only one replica.

### Health and metrics

`cloudflared` exposes its metrics and readiness endpoint on port `2000`:

```yaml
metrics: 0.0.0.0:2000
```

Kubernetes readiness and liveness probes query:

```text
http://<pod-ip>:2000/ready
```

### No in-container updates

Automatic updates are disabled:

```yaml
no-autoupdate: true
```

The container image is updated through the Kubernetes/GitOps workflow instead of allowing a running container to update itself.

## Configuration and credentials

The deployment uses two mounted data sources:

| Source | Container path | Purpose |
| --- | --- | --- |
| ConfigMap | `/etc/cloudflared/config/config.yaml` | Tunnel, metrics, and ingress configuration |
| Secret | `/etc/cloudflared/credentials/credentials.json` | Authentication for the locally managed tunnel |

The Secret is mounted as a directory. Its `credentials.json` key becomes a file inside that directory:

```text
/etc/cloudflared/
├── config/
│   └── config.yaml
└── credentials/
    └── credentials.json
```

The application configuration must point to the resulting file exactly:

```yaml
credentials-file: /etc/cloudflared/credentials/credentials.json
```

Kubernetes does not automatically tell an application where a Secret or ConfigMap was mounted. The path configured in `cloudflared` and the path produced by the volume mount must match character for character.

## Secrets management

Tunnel credentials are stored as a Kubernetes Secret encrypted with SOPS before being committed to Git. Flux decrypts the resource in the cluster during reconciliation.

The decrypted credential must never be committed, printed in documentation, or pasted into logs or pull requests. A valid locally managed tunnel credential file contains sensitive fields such as the tunnel secret and account identifier.

When checking the Secret, inspect its metadata and key names without decoding the value:

```bash
kubectl describe secret tunnel-credentials -n cloudflared
```

## GitOps lifecycle

Changes follow this path:

```text
Git commit
    ↓
Git remote
    ↓
Flux source reconciliation
    ↓
SOPS decryption
    ↓
Kustomize applies Kubernetes resources
    ↓
Deployment rollout
```

To request an immediate reconciliation after pushing a change:

```bash
flux reconcile kustomization infrastructure --with-source
```

Then watch the rollout:

```bash
kubectl rollout status deployment/cloudflared -n cloudflared
```

## Validation

### Check Flux

```bash
flux get kustomization infrastructure
```

The `infrastructure` Kustomization should report `READY=True` and show the expected Git revision.

### Check the Deployment and pods

```bash
kubectl get deployment cloudflared -n cloudflared
kubectl get pods -n cloudflared
```

Expected state:

```text
Deployment: 2/2 available
Pods:       1/1 Running, with no repeated restarts
```

### Check tunnel logs

```bash
kubectl logs -n cloudflared deployment/cloudflared
```

Healthy logs include messages such as:

```text
Starting tunnel
Registered tunnel connection
```

The startup settings should reference:

```text
/etc/cloudflared/config/config.yaml
/etc/cloudflared/credentials/credentials.json
```

### Check the tunnel from the Cloudflare CLI

```bash
cloudflared tunnel info homelab-staging
```

The output should show active connectors for the tunnel.

### Check DNS

```bash
dig mealie.olakunle.ca
dig linkding.olakunle.ca
```

The names must resolve through Cloudflare. `NXDOMAIN` means the public DNS record is absent or is not visible through the authoritative DNS zone.

### Check public routes

```bash
curl -I https://mealie.olakunle.ca
curl -I https://linkding.olakunle.ca
```

An application response such as `200`, `301`, or `302` confirms that DNS, the tunnel, hostname routing, and the origin Service are working together.

### Check an origin inside the cluster

If the tunnel is healthy but a public route returns an origin error, test the Service independently:

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  -it --rm -- \
  curl -I http://mealie-svc.mealie.svc.cluster.local:9000
```

This separates a Kubernetes Service problem from a Cloudflare Tunnel problem.

## Adding another application

Adding a new public application requires all of the following:

1. Confirm that the application has a working Kubernetes Service and EndpointSlice.
2. Add a hostname-to-Service rule above the final `http_status:404` rule.
3. Create the corresponding Cloudflare DNS route to `homelab-staging`.
4. Commit and push the manifest change.
5. Reconcile Flux and verify the rollout.
6. Test DNS and the public HTTPS endpoint.

Example ingress entry:

```yaml
- hostname: app.olakunle.ca
  service: http://app-svc.app.svc.cluster.local:8080
```

Example DNS command:

```bash
cloudflared tunnel route dns homelab-staging app.olakunle.ca
```

The final catch-all rule must remain last:

```yaml
- service: http_status:404
```

## Troubleshooting by layer

Follow the request path in order instead of changing multiple layers at once.

| Symptom | First layer to check |
| --- | --- |
| `NXDOMAIN` or `Could not resolve host` | Cloudflare DNS and authoritative nameservers |
| Tunnel has no active connectors | cloudflared pod logs, credentials, and outbound connectivity |
| `cloudflared` requests `cert.pem` unexpectedly | credential-file path and locally managed tunnel credentials |
| Cloudflare origin/gateway error | cloudflared ingress target and Kubernetes Service reachability |
| Service exists but has no backends | Service selector and EndpointSlice |
| One hostname reaches the wrong application | ingress hostname rules and their order |

Useful commands:

```bash
kubectl get pods -n cloudflared
kubectl logs -n cloudflared deployment/cloudflared
kubectl describe deployment cloudflared -n cloudflared
kubectl get svc -A
kubectl get endpointslice -A
cloudflared tunnel info homelab-staging
```

The `cloudflared` container image is intentionally minimal and may not include tools such as `cat`, `ls`, or a shell. A failed `kubectl exec ... -- cat` therefore does not prove that a mounted file is absent. Use the rendered Pod specification, application startup logs, or a suitable diagnostic container to inspect mounts safely.

## Debugging lesson: paths are contracts

The initial deployment failed because the credentials path expected by `cloudflared` did not match the path created by the Kubernetes mount.

The application and Kubernetes were each configured independently:

```text
cloudflared configuration → where the application looks
Kubernetes volumeMount    → where the file actually exists
```

The working mapping is:

```text
Secret key:          credentials.json
Directory mount:     /etc/cloudflared/credentials
Resulting file:      /etc/cloudflared/credentials/credentials.json
Application expects: /etc/cloudflared/credentials/credentials.json
```

For any future file-mount problem, compare these two values first:

```text
Application expects: ____________________________________
Kubernetes creates:  ____________________________________
```

Only move to application-specific authentication or networking theories after those paths match and the file is readable.

## Security considerations

- The tunnel creates outbound connections; it does not require inbound router port forwarding.
- Only explicitly configured hostnames are routed to applications.
- The catch-all rule returns `404` for unmatched traffic.
- Tunnel credentials remain encrypted in Git and are mounted read-only.
- Public reachability is not the same as authorization. Applications should retain their own authentication, and Cloudflare Access can be added where stronger edge-level access control is required.
- Kubernetes NetworkPolicies can further restrict which Services the `cloudflared` namespace may reach.
- Container image versions should be pinned and updated deliberately through GitOps for reproducible deployments.

## Scope

This component owns the tunnel connector and its hostname-to-origin routing. It does not own:

- the application Deployments or Services;
- the root `olakunle.ca` website or its CloudFront origin;
- application authentication and authorization; or
- the availability of an origin application after traffic reaches its Service.

Keeping those boundaries explicit makes failures easier to isolate and prevents unrelated DNS or application issues from being mistaken for tunnel failures.
