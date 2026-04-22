# kubernetes command ref

## concepts & components

### the big picture
A **cluster** is the whole Kubernetes environment. It has two parts:
- **Control Plane** — the brain. Schedules workloads, maintains desired state, handles API calls. In AKS, Microsoft manages this for you.
- **Node Pool** — a group of VMs (nodes) that actually run your workloads. You manage these.

### workload primitives
| Term | What it is |
|---|---|
| **Pod** | Smallest deployable unit. One or more containers sharing network/storage. Ephemeral — dies and gets replaced. |
| **Deployment** | Declares desired state: "keep 3 replicas of this pod running." Self-healing. |
| **ReplicaSet** | What a Deployment creates under the hood to maintain pod count. Rarely touched directly. |
| **StatefulSet** | Like a Deployment but for stateful apps (databases). Pods get stable, ordered identities. |
| **DaemonSet** | Runs exactly one pod on every node. Used for monitoring agents, log shippers, etc. |
| **CronJob** | Runs a pod on a schedule (cron syntax). One-off tasks via `Job`. |

### networking
| Term | What it is |
|---|---|
| **Service (ClusterIP)** | Default. Stable internal address for pods. Only reachable inside the cluster. |
| **Service (NodePort)** | Opens a port on every node VM for external access. Dev/test only, not production. |
| **Service (LoadBalancer)** | Provisions a cloud load balancer (Azure LB in AKS) with a public IP. One IP per service — gets expensive. |
| **Ingress** | Sits in front of services. Routes HTTP/HTTPS by hostname or path to the right service. One public IP for everything. NPM equivalent. |

### configuration & storage
| Term | What it is |
|---|---|
| **ConfigMap** | Stores non-sensitive config (env vars, config files) outside your container image. |
| **Secret** | Same as ConfigMap but for sensitive values. Base64 encoded (add Azure Key Vault for real encryption). |
| **PersistentVolume (PV)** | Actual storage provisioned in the cluster (Azure Disk or Azure Files in AKS). |
| **PersistentVolumeClaim (PVC)** | A pod's request for storage. Kubernetes matches it to a PV. Like a Docker volume mount. |

### scaling
| Term | What it is |
|---|---|
| **HPA (Horizontal Pod Autoscaler)** | Adds/removes pod replicas based on CPU or memory. Scales your app. |
| **Cluster Autoscaler** | Adds/removes nodes from the node pool based on scheduling pressure. Scales your infrastructure. |

### access & identity
| Term | What it is |
|---|---|
| **Namespace** | Logical folder inside a cluster. Isolates resources between teams or apps. Like a VLAN for workloads. |
| **RBAC** | Role-based access control. Who can do what to which resources in which namespace. |
| **ServiceAccount** | Identity assigned to a pod. Used for RBAC and (in AKS) Managed Identity for Azure resource access. |
| **Managed Identity** | AKS-specific. Pods authenticate to Azure services (Key Vault, ACR, Storage) without any credentials. |

### aks-specific
| Term | What it is |
|---|---|
| **ACR (Azure Container Registry)** | Azure's private Docker registry. AKS pulls images from here natively. |
| **System Node Pool** | Required node pool that runs AKS system pods (DNS, metrics, etc). Don't run your apps here. |
| **User Node Pool** | Where your application workloads run. Can have multiple with different VM sizes. |
| **Azure CNI** | Production networking mode. Pods get real VNet IPs — enables hybrid/on-prem connectivity. |
| **Kubenet** | Simpler networking mode. Pods get overlay IPs, nodes get VNet IPs. Fine for isolated clusters. |

### manifest structure (yaml)
Everything in Kubernetes is declared in YAML. The four required fields:

```yaml
apiVersion: apps/v1   # which API handles this resource
kind: Deployment      # what type of resource
metadata:             # name, namespace, labels
spec:                 # what you actually want
```

---

## kubectl setup

```bash
# install kubectl (macOS)
brew install kubectl

# install kubectl (Linux)
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && mv kubectl /usr/local/bin/

# verify
kubectl version --client

# shell completion (bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# useful alias (standard convention)
alias k=kubectl
complete -F __start_kubectl k
```

## contexts & config

```bash
kubectl config view                          # show kubeconfig
kubectl config get-contexts                  # list all contexts
kubectl config current-context               # show active context
kubectl config use-context <n>               # switch context
kubectl config set-context --current --namespace=<ns>  # set default namespace
kubectl config delete-context <n>            # remove context
kubectl cluster-info                         # show cluster endpoints
kubectl cluster-info dump                    # full cluster debug dump
```

## namespaces

```bash
kubectl get namespaces                       # list namespaces
kubectl create namespace <n>                 # create namespace
kubectl delete namespace <n>                 # delete namespace (and all resources)
kubectl config set-context --current --namespace=<ns>  # set default ns
# use -n flag to target a namespace
kubectl get pods -n kube-system
kubectl get all -n <ns>                      # all resources in namespace
kubectl get all --all-namespaces             # all resources across all namespaces
```

## pods

```bash
kubectl get pods                             # list pods (current ns)
kubectl get pods -A                          # all namespaces
kubectl get pods -o wide                     # with node and IP info
kubectl get pods -w                          # watch for changes
kubectl get pod <n> -o yaml                  # full YAML spec
kubectl describe pod <n>                     # detailed info + events
kubectl logs <n>                             # pod logs
kubectl logs <n> -f                          # follow logs
kubectl logs <n> --tail=100                  # last 100 lines
kubectl logs <n> -c <container>              # specific container logs
kubectl logs <n> --previous                  # logs from previous (crashed) container
kubectl exec -it <n> -- bash                 # shell into pod
kubectl exec -it <n> -c <c> -- bash         # shell into specific container
kubectl exec <n> -- <cmd>                    # run command in pod
kubectl cp <n>:/path/file ./file             # copy from pod
kubectl cp ./file <n>:/path/                 # copy to pod
kubectl delete pod <n>                       # delete pod (recreated if managed)
kubectl delete pod <n> --force               # force delete (stuck terminating)
kubectl port-forward pod/<n> 8080:80         # forward local port to pod
kubectl top pod                              # CPU/memory usage (needs metrics-server)
kubectl top pod <n>                          # specific pod
```

## deployments

```bash
kubectl get deployments                      # list deployments
kubectl get deploy <n> -o yaml               # full spec
kubectl describe deploy <n>                  # detailed info
kubectl create deployment <n> --image=<img>  # create deployment
kubectl apply -f deployment.yaml             # apply from file
kubectl delete deployment <n>               # delete deployment
kubectl scale deployment <n> --replicas=3    # scale
kubectl rollout status deployment/<n>        # watch rollout progress
kubectl rollout history deployment/<n>       # rollout history
kubectl rollout undo deployment/<n>          # rollback to previous
kubectl rollout undo deployment/<n> --to-revision=2  # rollback to revision
kubectl rollout restart deployment/<n>       # rolling restart (triggers new rollout)
kubectl set image deployment/<n> <container>=<image>:<tag>  # update image
kubectl autoscale deployment <n> --min=2 --max=5 --cpu-percent=80  # HPA
```

## services

```bash
kubectl get services                         # list services
kubectl get svc                              # short form
kubectl describe svc <n>                     # service details
kubectl expose deployment <n> --port=80 --type=ClusterIP   # expose deployment
kubectl expose deployment <n> --port=80 --type=NodePort     # node port
kubectl expose deployment <n> --port=80 --type=LoadBalancer # cloud LB
kubectl delete svc <n>                       # delete service
kubectl port-forward svc/<n> 8080:80         # forward local port to service
```

## nodes

```bash
kubectl get nodes                            # list nodes
kubectl get nodes -o wide                    # with IPs and OS info
kubectl describe node <n>                    # detailed node info
kubectl top node                             # CPU/memory per node
kubectl cordon <n>                           # mark unschedulable
kubectl uncordon <n>                         # mark schedulable again
kubectl drain <n> --ignore-daemonsets        # evict pods (prep for maintenance)
kubectl label node <n> key=value             # add label to node
kubectl taint node <n> key=value:NoSchedule  # add taint
```

## configmaps & secrets

```bash
kubectl get configmaps                       # list configmaps
kubectl get cm <n> -o yaml                   # view configmap
kubectl create configmap <n> --from-file=./config.conf   # from file
kubectl create configmap <n> --from-literal=key=value    # inline
kubectl delete configmap <n>                 # delete

kubectl get secrets                          # list secrets
kubectl get secret <n> -o yaml               # view secret (base64 encoded)
kubectl create secret generic <n> --from-literal=key=value
kubectl create secret docker-registry <n> \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
kubectl delete secret <n>

# decode a secret value
kubectl get secret <n> -o jsonpath="{.data.key}" | base64 -d
```

## resource management

```bash
# apply / delete
kubectl apply -f file.yaml                   # create or update
kubectl apply -f ./dir/                      # apply all files in dir
kubectl apply -k ./kustomize/               # apply kustomize dir
kubectl delete -f file.yaml                  # delete resources defined in file
kubectl delete pod,svc --all -n <ns>         # delete all pods and services in ns

# get / describe (works for any resource type)
kubectl get <resource>                       # list
kubectl get <resource> <n>                   # specific
kubectl get <resource> -l key=value          # filter by label
kubectl describe <resource> <n>              # details + events

# short resource names
# po=pods, deploy=deployments, svc=services, cm=configmaps
# ns=namespaces, no=nodes, pv=persistentvolumes, pvc=persistentvolumeclaims
# ing=ingresses, sa=serviceaccounts, rs=replicasets, sts=statefulsets, ds=daemonsets

kubectl get po,svc,deploy -n <ns>            # multiple resource types at once
kubectl get all -n <ns>                      # common resources (not truly all)
```

## labels & selectors

```bash
kubectl label pod <n> env=prod               # add label
kubectl label pod <n> env-                   # remove label
kubectl get pods -l env=prod                 # filter by label
kubectl get pods -l 'env in (prod,staging)'  # set-based selector
kubectl get pods --show-labels               # show all labels
kubectl annotate pod <n> description="..."   # add annotation
```

## resource yaml structure

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
    env: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myimage:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "db.default.svc.cluster.local"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: db_password
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /etc/config
      volumes:
        - name: config
          configMap:
            name: myapp-config
```

## common resource types

```yaml
# service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP

---
# ingress (nginx)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80

---
# persistent volume claim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard

---
# cronjob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: busybox
              command: ["/bin/sh", "-c", "echo cleaning up"]
          restartPolicy: OnFailure
```

## statefulsets & daemonsets

```bash
kubectl get statefulsets                     # list statefulsets
kubectl get sts <n> -o yaml                  # view spec
kubectl scale sts <n> --replicas=3           # scale statefulset
kubectl rollout restart sts/<n>              # rolling restart

kubectl get daemonsets                       # list daemonsets (one pod per node)
kubectl describe ds <n>                      # daemonset details
```

## persistent volumes

```bash
kubectl get pv                               # list persistent volumes (cluster-wide)
kubectl get pvc                              # list persistent volume claims (ns-scoped)
kubectl describe pv <n>                      # PV details
kubectl describe pvc <n>                     # PVC details
kubectl delete pvc <n>                       # delete PVC
```

## rbac

```bash
kubectl get roles -n <ns>                    # list roles
kubectl get rolebindings -n <ns>             # list role bindings
kubectl get clusterroles                     # cluster-wide roles
kubectl get clusterrolebindings              # cluster-wide bindings
kubectl describe role <n> -n <ns>            # role details
kubectl auth can-i <verb> <resource>         # check own permissions
kubectl auth can-i get pods --as=user@example.com  # check as another user
kubectl auth can-i --list                    # list all permissions
```

## debugging & troubleshooting

```bash
# why is a pod not starting?
kubectl describe pod <n>                     # check Events section
kubectl logs <n> --previous                  # logs from crashed container

# run a temporary debug pod
kubectl run debug --image=busybox --rm -it --restart=Never -- sh
kubectl run debug --image=nicolaka/netshoot --rm -it --restart=Never -- bash

# check DNS resolution from inside cluster
kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default

# check resource usage
kubectl top pods --sort-by=memory
kubectl top pods --sort-by=cpu

# get events sorted by time
kubectl get events --sort-by='.lastTimestamp' -n <ns>

# watch events in real time
kubectl get events -w -n <ns>

# check what's using a PVC
kubectl get pods -o json | jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="my-pvc") | .metadata.name'

# force delete stuck terminating pod
kubectl delete pod <n> --force --grace-period=0
```

## kustomize

```bash
kubectl apply -k ./dir/                      # apply kustomize directory
kubectl kustomize ./dir/                     # preview rendered output
```

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

namespace: myapp

commonLabels:
  app: myapp
  env: prod

images:
  - name: myimage
    newTag: "1.2.3"

configMapGenerator:
  - name: myapp-config
    files:
      - config.conf
```

## helm

```bash
helm repo add stable https://charts.helm.sh/stable   # add repo
helm repo update                             # update all repos
helm repo list                               # list repos
helm search repo <chart>                     # search charts
helm install <release> <chart>               # install chart
helm install <release> <chart> -f values.yaml  # with custom values
helm install <release> <chart> --set key=value  # inline value override
helm upgrade <release> <chart>               # upgrade release
helm upgrade --install <release> <chart>     # install or upgrade
helm list                                    # list releases
helm list -A                                 # all namespaces
helm status <release>                        # release status
helm get values <release>                    # show applied values
helm get manifest <release>                  # show rendered manifests
helm rollback <release> <revision>           # rollback to revision
helm uninstall <release>                     # uninstall release
helm template <release> <chart> -f vals.yaml # render templates locally
helm lint ./chart/                           # lint a chart
```

## useful one-liners

```bash
# watch all pods across all namespaces
watch kubectl get pods -A

# restart all deployments in a namespace
kubectl rollout restart deployment -n <ns>

# get all images running in cluster
kubectl get pods -A -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s ' ' '\n' | sort -u

# list all resources in a namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I{} kubectl get {} -n <ns> --ignore-not-found 2>/dev/null

# scale all deployments to 0 in a namespace (maintenance mode)
kubectl scale deployment --all --replicas=0 -n <ns>

# get pod by label and exec into it
kubectl exec -it $(kubectl get pod -l app=myapp -o jsonpath='{.items[0].metadata.name}') -- bash

# copy a secret between namespaces
kubectl get secret <n> -n source -o yaml | \
  sed 's/namespace: source/namespace: dest/' | \
  kubectl apply -n dest -f -
```

---

## eks (aws)

### aks vs eks — key differences
| | AKS | EKS |
|---|---|---|
| **Control plane cost** | Free | ~$0.10/hr per cluster |
| **Control plane mgmt** | Fully managed by Microsoft | Managed by AWS, but you pay |
| **Default networking** | Kubenet or Azure CNI | AWS VPC CNI (pods get real VPC IPs) |
| **Node groups** | Node Pools | Node Groups (managed) or Fargate |
| **Identity/auth** | Managed Identity + Azure AD | IAM Roles for Service Accounts (IRSA) |
| **Private registry** | ACR | ECR (Elastic Container Registry) |
| **Ingress** | AGIC or NGINX | AWS Load Balancer Controller or NGINX |
| **Observability** | Azure Monitor + Container Insights | CloudWatch Container Insights |
| **Upgrades** | In-place rolling upgrade | Managed node group rolling update |

### eks-specific concepts
| Term | What it is |
|---|---|
| **Node Group** | EKS equivalent of AKS node pool. Group of EC2 instances running your pods. |
| **Fargate** | Serverless pods — no nodes to manage at all. AWS provisions compute per pod. |
| **ECR (Elastic Container Registry)** | AWS private Docker registry. EKS equivalent of ACR. |
| **IRSA (IAM Roles for Service Accounts)** | Pods authenticate to AWS services via IAM role. EKS equivalent of Managed Identity. |
| **aws-auth ConfigMap** | Maps IAM users/roles to Kubernetes RBAC. How you grant AWS users cluster access. |
| **eksctl** | CLI tool for creating/managing EKS clusters. Like `az aks` but for AWS. |
| **AWS Load Balancer Controller** | Provisions ALB/NLB from Kubernetes Ingress/Service definitions. |
| **VPC CNI** | Default EKS networking. Every pod gets a real VPC IP — same as Azure CNI in AKS. |
| **Karpenter** | Modern node autoscaler for EKS. More flexible than Cluster Autoscaler. |

### eksctl — cluster management

```bash
# install eksctl (macOS)
brew tap weaveworks/tap && brew install weaveworks/tap/eksctl

# install eksctl (Linux)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin/

# verify
eksctl version

# create a basic cluster
eksctl create cluster --name my-cluster --region us-east-1 --nodes 2

# create cluster from config file
eksctl create cluster -f cluster.yaml

# list clusters
eksctl get cluster

# delete cluster
eksctl delete cluster --name my-cluster --region us-east-1

# update kubeconfig to connect kubectl to EKS
aws eks update-kubeconfig --name my-cluster --region us-east-1

# get cluster info
eksctl get cluster --name my-cluster --region us-east-1
```

### node groups

```bash
# list node groups
eksctl get nodegroup --cluster my-cluster

# create a node group
eksctl create nodegroup \
  --cluster my-cluster \
  --name my-nodegroup \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4

# scale a node group
eksctl scale nodegroup \
  --cluster my-cluster \
  --name my-nodegroup \
  --nodes 3

# delete a node group
eksctl delete nodegroup --cluster my-cluster --name my-nodegroup

# upgrade node group AMI
eksctl upgrade nodegroup --cluster my-cluster --name my-nodegroup
```

### aws cli — eks commands

```bash
# list clusters
aws eks list-clusters --region us-east-1

# describe cluster
aws eks describe-cluster --name my-cluster --region us-east-1

# update kubeconfig
aws eks update-kubeconfig --name my-cluster --region us-east-1

# list node groups
aws eks list-nodegroups --cluster-name my-cluster

# describe node group
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name my-nodegroup

# list Fargate profiles
aws eks list-fargate-profiles --cluster-name my-cluster

# get available k8s versions
aws eks describe-addon-versions --query 'addons[].addonVersions[].compatibilities[].clusterVersion' \
  --output text | tr '\t' '\n' | sort -u
```

### irsa — iam roles for service accounts

```bash
# enable OIDC provider for cluster (required for IRSA)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region us-east-1 \
  --approve

# create IAM service account with policy attached
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace default \
  --name my-service-account \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# list IAM service accounts
eksctl get iamserviceaccount --cluster my-cluster
```

### ecr — elastic container registry

```bash
# authenticate docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com

# create a repo
aws ecr create-repository --repository-name my-app --region us-east-1

# list repos
aws ecr describe-repositories --region us-east-1

# tag and push image
docker tag my-app:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# list images in repo
aws ecr list-images --repository-name my-app --region us-east-1
```

### eks cluster config (yaml)

```yaml
# cluster.yaml — eksctl cluster definition
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: "1.29"

managedNodeGroups:
  - name: general
    instanceType: t3.medium
    minSize: 2
    maxSize: 5
    desiredCapacity: 2
    volumeSize: 50
    ssh:
      allow: false
    labels:
      role: general
    tags:
      Environment: production

  - name: high-memory
    instanceType: r5.xlarge
    minSize: 1
    maxSize: 3
    desiredCapacity: 1
    labels:
      role: high-memory

iam:
  withOIDC: true  # required for IRSA
```

### aws-auth configmap — cluster access

```bash
# view current aws-auth configmap
kubectl describe configmap aws-auth -n kube-system

# edit to add IAM user or role access
kubectl edit configmap aws-auth -n kube-system
```

```yaml
# aws-auth configmap structure
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::<account-id>:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::<account-id>:role/AdminRole
      username: admin
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::<account-id>:user/myuser
      username: myuser
      groups:
        - system:masters
```

### debugging eks-specific issues

```bash
# check node group instance health
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name my-nodegroup \
  --query 'nodegroup.health'

# check cluster add-ons
aws eks list-addons --cluster-name my-cluster
aws eks describe-addon --cluster-name my-cluster --addon-name vpc-cni

# update an add-on
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni \
  --addon-version v1.16.0-eksbuild.1

# check VPC CNI logs (networking issues)
kubectl logs -n kube-system -l k8s-app=aws-node

# check CoreDNS (DNS issues)
kubectl logs -n kube-system -l k8s-app=kube-dns

# check nodes can reach API server
kubectl get --raw /healthz

# check IRSA is working for a pod
kubectl exec -it <pod> -- aws sts get-caller-identity
```

---

## vanilla kubernetes (kubeadm)

### what you own vs managed k8s
In AKS/EKS the control plane is managed for you. With vanilla kubeadm you own **everything**:
- Control plane nodes (API server, etcd, scheduler, controller-manager)
- etcd backups
- Certificate rotation
- Kubernetes version upgrades
- CNI plugin installation and management
- Load balancer integration (MetalLB for bare metal)

### prerequisites (all nodes)

```bash
# disable swap (required by kubelet)
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# load required kernel modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# set sysctl params
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# install containerd
apt install -y containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
# set SystemdCgroup = true in config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# install kubeadm, kubelet, kubectl
apt-get update
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

### bootstrap the control plane (first master node only)

```bash
# initialize cluster
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \        # match your CNI (Flannel uses this)
  --control-plane-endpoint=<lb-or-vip>:6443 \ # use a VIP if HA
  --upload-certs

# set up kubeconfig for current user
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# install CNI plugin (pick one — Flannel shown)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# for Calico instead
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# for Cilium instead
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system
```

### join worker nodes

```bash
# on the control plane — generate join command
kubeadm token create --print-join-command

# run the output on each worker node, e.g.:
kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# join an additional control plane node (HA)
kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>
```

### cni plugins — comparison
| CNI | Best for | Notes |
|---|---|---|
| **Flannel** | Simple homelabs, getting started | Basic overlay, no network policy |
| **Calico** | Production, network policy needed | eBPF mode available, widely used |
| **Cilium** | Advanced, eBPF-native | Best observability, Hubble UI, replaces kube-proxy |
| **Weave** | Simple multi-host | Being deprecated, avoid for new clusters |

### etcd — backup & restore

```bash
# install etcdctl
apt install etcd-client

# snapshot backup (run on control plane node)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-$(date +%Y%m%d).db --write-out=table

# restore from snapshot (stop kube-apiserver first)
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20240101.db \
  --data-dir=/var/lib/etcd-restore
# then update etcd manifest to point to new data-dir
```

### certificate management

```bash
# check cert expiry (all certs)
kubeadm certs check-expiration

# renew all certs (do this before they expire — default 1 year)
kubeadm certs renew all

# restart control plane components after renewal
kubectl -n kube-system rollout restart deployment/coredns
# control plane static pods restart automatically after cert renewal
```

### cluster upgrades

```bash
# check available versions
apt-cache madison kubeadm

# upgrade kubeadm first
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.30.0-1.1
apt-mark hold kubeadm

# plan the upgrade (shows what will change)
kubeadm upgrade plan

# apply the upgrade (control plane only)
kubeadm upgrade apply v1.30.0

# upgrade kubelet and kubectl on each node
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet

# drain node before upgrading worker
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# upgrade kubelet on worker, then:
kubectl uncordon <node>
```

### control plane health checks

```bash
# check component status
kubectl get componentstatuses

# check control plane pods
kubectl get pods -n kube-system

# check API server directly
curl -k https://localhost:6443/healthz

# check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# view static pod manifests (control plane config lives here)
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml, kube-controller-manager.yaml, kube-scheduler.yaml, etcd.yaml
```

### metallb — load balancer for bare metal

```bash
# without a cloud provider there is no LoadBalancer type — MetalLB fills that gap
# install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml

# configure IP address pool
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.70.200-192.168.70.220   # adjust to your lab subnet
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
EOF
```
