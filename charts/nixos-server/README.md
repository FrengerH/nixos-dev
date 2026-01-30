# NixOS Server Helm Chart

A minimal Helm chart for deploying a headless NixOS server pod to Kubernetes with persistent storage for Nix store and home directory, with optional SSH access.

## Installation

```bash
helm install nixos-server ./nixos-server
```

## Configuration

The following table lists the configurable parameters of the NixOS server chart:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Container image repository | `nixos/nix` |
| `image.tag` | Container image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `nameOverride` | String to partially override `fullname` template | `""` |
| `fullnameOverride` | String to fully override `fullname` template | `""` |
| `podAnnotations` | Annotations to add to the pod | `{}` |
| `securityContext` | Pod security context | See values.yaml |
| `resources` | CPU/Memory resource requests/limits | See values.yaml |
| `nodeSelector` | Node labels for pod assignment | `{}` |
| `tolerations` | Tolerations for pod assignment | `[]` |
| `affinity` | Affinity rules for pod assignment | `{}` |
| `command` | Container entrypoint command | `["tail", "-f", "/dev/null"]` |
| `args` | Container arguments | `[]` |
| `env` | Additional environment variables | `[]` |
| `persistence.enabled` | Enable persistent volumes | `true` |
| `persistence.nixStore.enabled` | Enable Nix store persistence | `true` |
| `persistence.nixStore.mountPath` | Nix store mount path | `/nix` |
| `persistence.nixStore.size` | Nix store PVC size | `10Gi` |
| `persistence.nixStore.storageClass` | Nix store storage class | `""` |
| `persistence.homeDir.enabled` | Enable home directory persistence | `true` |
| `persistence.homeDir.mountPath` | Home directory mount path | `/home/nix` |
| `persistence.homeDir.size` | Home directory PVC size | `5Gi` |
| `persistence.homeDir.storageClass` | Home directory storage class | `""` |
| `ssh.enabled` | Enable SSH server | `true` |
| `ssh.port` | SSH port | `22` |
| `ssh.username` | SSH username | `nix` |
| `ssh.password` | SSH password (empty = keys only) | `""` |
| `ssh.authorizedKeys` | List of SSH public keys | `[]` |
| `service.type` | Service type (LoadBalancer/NodePort/ClusterIP) | `LoadBalancer` |
| `service.annotations` | Service annotations | `{}` |
| `ingress.enabled` | Enable Ingress | `false` |
| `ingress.className` | Ingress class name | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Ingress host configuration | See values.yaml |
| `ingress.tls` | Ingress TLS configuration | `[]` |

## Usage

### Basic Deployment

```bash
helm install nixos-server ./nixos-server
```

### Custom Command

Create a `values-custom.yaml` file:

```yaml
command:
  - /bin/sh
  - -c
  - "while true; do sleep 30; done"
```

Install with custom values:

```bash
helm install nixos-server ./nixos-server -f values-custom.yaml
```

### With Environment Variables

```yaml
env:
  - name: MY_VAR
    value: "my-value"
```

### Configuring Persistence

By default, the chart creates two persistent volumes:
- `/nix` - Nix store (10Gi) for persisting installed packages
- `/home/nix` - Home directory (5Gi) for user files

To disable persistence:

```yaml
persistence:
  enabled: false
```

To customize storage size or use a specific storage class:

```yaml
persistence:
  nixStore:
    size: 20Gi
    storageClass: fast-ssd
  homeDir:
    size: 10Gi
    storageClass: standard
```

To disable specific volumes:

```yaml
persistence:
  enabled: true
  nixStore:
    enabled: false
  homeDir:
    enabled: true
```

### SSH Access

By default, SSH is enabled with the following configuration:
- Username: `nix`
- Password: none (SSH keys only)
- Port: 22
- Service type: `LoadBalancer`

#### SSH with Password

```yaml
ssh:
  password: "your-secure-password"
```

#### SSH with Public Keys (Recommended)

```yaml
ssh:
  password: ""
  authorizedKeys:
    - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC..."
    - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAV..."
```

#### Service Configuration

To use NodePort instead of LoadBalancer:

```yaml
service:
  type: NodePort
```

To use ClusterIP with port forwarding:

```yaml
service:
  type: ClusterIP
```

Then forward the port:
```bash
kubectl port-forward svc/nixos-server 2222:22
```

#### Connecting to SSH

With LoadBalancer service:
```bash
# Get the external IP
kubectl get svc nixos-server

# SSH to the service
ssh nix@<EXTERNAL_IP>
```

With NodePort:
```bash
# Get the node port
kubectl get svc nixos-server

# SSH to any node
ssh -p <NODE_PORT> nix@<NODE_IP>
```

With kubectl port-forward:
```bash
# In one terminal
kubectl port-forward svc/nixos-server 2222:22

# In another terminal
ssh -p 2222 nix@localhost
```

### Ingress Configuration

Note: Standard Kubernetes Ingress is designed for HTTP/HTTPS traffic, not SSH. If you need to expose SSH externally, use LoadBalancer or NodePort services as shown above.

For HTTP-based services, enable Ingress:

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: nixos.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: nixos-tls
      hosts:
        - nixos.example.com
```

## Interacting with the Pod

```bash
kubectl exec -it deployment/nixos-server -- /bin/sh
```

## Uninstalling

```bash
helm uninstall nixos-server
```