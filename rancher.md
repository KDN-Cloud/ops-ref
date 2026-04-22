# rancher reference

## what rancher is
Rancher is a Kubernetes management platform that sits on top of one or more clusters. It gives you:
- A web UI to manage multiple clusters from one place
- Built-in user management (local, GitHub, LDAP/AD, Okta)
- Cluster provisioning (spin up RKE2/k3s/EKS/AKS clusters from the UI)
- App catalog (Helm charts via UI)
- RBAC across clusters
- Monitoring (Prometheus/Grafana) and logging built in as installable apps

Think of it as the control plane for your control planes.

## rancher vs rke2 vs k3s
| | What it is |
|---|---|
| **Rancher** | The management UI/platform. Needs a Kubernetes cluster to run on. |
| **RKE2** | Rancher's production Kubernetes distro. FIPS-compliant, hardened. Replaces RKE1. |
| **k3s** | Lightweight Kubernetes for edge/homelab. Single binary, minimal resources. |
| **RKE1** | Original Rancher Kubernetes Engine. Legacy — prefer RKE2 for new clusters. |

Common homelab setup: **k3s** or **RKE2** as the underlying cluster, **Rancher** running on top managing it and any additional clusters.

---

## option 1 — rancher on k3s (recommended for homelab)

```bash
# step 1: install k3s (single node — control plane + worker)
curl -sfL https://get.k3s.io | sh -

# verify k3s is running
systemctl status k3s
kubectl get nodes

# get kubeconfig
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# step 2: install cert-manager (required by Rancher)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# wait for cert-manager to be ready
kubectl rollout status deployment/cert-manager -n cert-manager
kubectl rollout status deployment/cert-manager-webhook -n cert-manager

# step 3: add Rancher Helm repo
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

# step 4: install Rancher
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com \        # your DNS entry
  --set bootstrapPassword=admin \             # change this immediately after login
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com \
  --set letsEncrypt.ingress.class=traefik    # k3s uses traefik by default

# watch rollout
kubectl rollout status deployment/rancher -n cattle-system

# step 5: get the bootstrap password if you didn't set one
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}'
```

---

## option 2 — rancher on rke2 (production-grade)

```bash
# step 1: install RKE2 server (control plane node)
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service

# get kubeconfig
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
# kubectl is at:
/var/lib/rancher/rke2/bin/kubectl

# get node join token
cat /var/lib/rancher/rke2/server/node-token

# step 2: join worker nodes
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service

# configure agent
mkdir -p /etc/rancher/rke2
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://<control-plane-ip>:9345
token: <node-token>
EOF

systemctl start rke2-agent.service

# step 3: install Rancher on top (same Helm steps as k3s above)
# but use nginx ingress class instead of traefik:
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com
```

---

## adding clusters to rancher

Once Rancher is running you can import or provision clusters from the UI:

```
Rancher UI → Cluster Management → Create

Options:
- Custom (import existing cluster — paste kubectl apply command on target cluster)
- Amazon EKS (provision directly from Rancher using AWS creds)
- Azure AKS (provision directly from Rancher using Azure creds)
- RKE2/k3s (Rancher provisions nodes via SSH)
```

```bash
# import an existing cluster (run on the target cluster)
# Rancher gives you this command — it looks like:
kubectl apply -f https://rancher.example.com/v3/import/<token>.yaml
```

---

## rancher cli

```bash
# install rancher cli
curl -sfL https://github.com/rancher/cli/releases/latest/download/rancher-linux-amd64-v2.8.0.tar.gz | tar xz
mv rancher /usr/local/bin/

# login
rancher login https://rancher.example.com --token <api-token>

# list clusters
rancher clusters ls

# switch context to a cluster
rancher context switch

# list namespaces
rancher namespaces ls

# list workloads
rancher ps
```

---

## rancher useful urls
| Path | What it is |
|---|---|
| `/dashboard` | Main UI |
| `/dashboard/c/<cluster>/explorer` | Cluster Explorer (kubectl-like UI) |
| `/v3` | Rancher API v3 |
| `/v1` | Steve API (newer) |
| `<rancher-url>/v3/import/<token>.yaml` | Import manifest for existing cluster |

---

## k3s — standalone (without rancher)

```bash
# install k3s single node
curl -sfL https://get.k3s.io | sh -

# install k3s with specific version
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.0+k3s1 sh -

# install k3s agent (worker node)
curl -sfL https://get.k3s.io | K3S_URL=https://<server>:6443 K3S_TOKEN=<token> sh -

# get node token (on server)
cat /var/lib/rancher/k3s/server/node-token

# kubeconfig location
/etc/rancher/k3s/k3s.yaml

# copy to local machine
scp root@<server>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
# replace 127.0.0.1 with server IP in the file

# check k3s status
systemctl status k3s
k3s kubectl get nodes

# uninstall k3s (server)
/usr/local/bin/k3s-uninstall.sh

# uninstall k3s (agent)
/usr/local/bin/k3s-agent-uninstall.sh

# k3s comes with these built in (no need to install separately):
# - traefik (ingress)
# - local-path provisioner (storage)
# - CoreDNS
# - Flannel (CNI)
# - MetalLB (optional, enable with flag)
```
