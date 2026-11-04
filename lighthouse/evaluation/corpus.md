# Homelab reference corpus

## Ownership

Infrastructure owns the live K3s cluster, provisioning scripts, cluster runtime
configuration, and local-only secrets. Lighthouse owns application backend code.
Chartroom owns the application interface. Fleet owns application admission and
release configuration. Shipyard owns application builds, tests, and image promotion.

## Ingress

Public traffic follows Cloudflare Edge -> cloudflared -> Traefik -> Kubernetes
Ingress -> ClusterIP Service -> ready workload. The NanoPi R6S is the upstream
router and firewall, not a Kubernetes node. No inbound ports are forwarded.
Cloudflared establishes an outbound tunnel.

Standard public hostnames use the nnvuf.uk domain. Public ingress uses the
cloudflare-origin-tls secret and HTTPS redirect middleware.

## Flux

The Flux bootstrap root owns only flux-system and gitops. Independent group
Kustomizations own platform, applications, and observability. Application and
observability groups depend on platform storage and middleware.

Helm owns Traefik, Authentik, and the Prometheus stack. Flux health checks must
target only resources Flux directly creates.

## Nodes

The mandatory control plane runs K3s in Windows WSL2. Its GPU is an NVIDIA
RTX 5070. It has AI, GPU, workloads, and monitoring placement labels.
The Orange Pi AIpro is an optional worker with AI and workloads labels.

GPU Ollama requests one nvidia.com/gpu resource. CPU Ollama requires an AI node
without the GPU label. Application workloads use soft placement affinity.

## Storage and networking

Stateful services use local-path storage with ReadWriteOnce access.
Persistent data is tied to the node hosting the volume.

Flannel is the active CNI. Committed NetworkPolicy manifests do not enforce
isolation until a policy-capable CNI is installed.

## Observability

Promtail reads Kubernetes container logs from /var/log/pods and parses them with
the CRI pipeline. Traefik exposes Prometheus metrics on port 9100.
Prometheus, Grafana, Loki, and Promtail provide cluster observability.

## External AI

The Mac mini is not a Kubernetes node. Native Ollama is reached through the
ollama-macos ClusterIP Service and its GitOps-managed EndpointSlice.
The address and port come from local-only cluster-ai-vars substitutions.

## Operations

Deployment phases run Helm charts first, then Cloudflare, then manifests, and
finally Flux bootstrap. Secrets are created imperatively and are never committed
as plaintext. Live cluster mutations require explicit authorization.
