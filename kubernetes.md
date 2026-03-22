# kubernetes command ref

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
kubectl config use-context <n>           # switch context
kubectl config set-context --current --namespace=<ns>  # set default namespace
kubectl config delete-context <n>        # remove context
kubectl cluster-info                         # show cluster endpoints
kubectl cluster-info dump                    # full cluster debug dump
```

## namespaces

```bash
kubectl get namespaces                       # list namespaces
kubectl create namespace <n>             # create namespace
kubectl delete namespace <n>             # delete namespace (and all resources)
kubectl config set-context --current --namespace=<ns>  # set default ns
# use -n flag to target a namespace
kubectl get pods -n kube-system
kubectl get all -n <ns>                  # all resources in namespace
kubectl get all --all-namespaces         # all resources across all namespaces
```

## pods

```bash
kubectl get pods                             # list pods (current ns)
kubectl get pods -A                          # all namespaces
kubectl get pods -o wide                     # with node and IP info
kubectl get pods -w                          # watch for changes
kubectl get pod <n> -o yaml               # full YAML spec
kubectl describe pod <n>                  # detailed info + events
kubectl logs <n>                          # pod logs
kubectl logs <n> -f                       # follow logs
kubectl logs <n> --tail=100              # last 100 lines
kubectl logs <n> -c <container>          # specific container logs
kubectl logs <n> --previous              # logs from previous (crashed) container
kubectl exec -it <n> -- bash             # shell into pod
kubectl exec -it <n> -c <c> -- bash     # shell into specific container
kubectl exec <n> -- <cmd>               # run command in pod
kubectl cp <n>:/path/file ./file         # copy from pod
kubectl cp ./file <n>:/path/             # copy to pod
kubectl delete pod <n>                   # delete pod (recreated if managed)
kubectl delete pod <n> --force           # force delete (stuck terminating)
kubectl port-forward pod/<n> 8080:80     # forward local port to pod
kubectl top pod                              # CPU/memory usage (needs metrics-server)
kubectl top pod <n>                       # specific pod
```

## deployments

```bash
kubectl get deployments                      # list deployments
kubectl get deploy <n> -o yaml            # full spec
kubectl describe deploy <n>               # detailed info
kubectl create deployment <n> --image=<img>  # create deployment
kubectl apply -f deployment.yaml             # apply from file
kubectl delete deployment <n>             # delete deployment
kubectl scale deployment <n> --replicas=3 # scale
kubectl rollout status deployment/<n>     # watch rollout progress
kubectl rollout history deployment/<n>    # rollout history
kubectl rollout undo deployment/<n>       # rollback to previous
kubectl rollout undo deployment/<n> --to-revision=2  # rollback to revision
kubectl rollout restart deployment/<n>    # rolling restart (triggers new rollout)
kubectl set image deployment/<n> <container>=<image>:<tag>  # update image
kubectl autoscale deployment <n> --min=2 --max=5 --cpu-percent=80  # HPA
```

## services

```bash
kubectl get services                         # list services
kubectl get svc                              # short form
kubectl describe svc <n>                  # service details
kubectl expose deployment <n> --port=80 --type=ClusterIP   # expose deployment
kubectl expose deployment <n> --port=80 --type=NodePort     # node port
kubectl expose deployment <n> --port=80 --type=LoadBalancer # cloud LB
kubectl delete svc <n>                    # delete service
kubectl port-forward svc/<n> 8080:80      # forward local port to service
```

## nodes

```bash
kubectl get nodes                            # list nodes
kubectl get nodes -o wide                    # with IPs and OS info
kubectl describe node <n>                 # detailed node info
kubectl top node                             # CPU/memory per node
kubectl cordon <n>                        # mark unschedulable
kubectl uncordon <n>                      # mark schedulable again
kubectl drain <n> --ignore-daemonsets     # evict pods (prep for maintenance)
kubectl label node <n> key=value          # add label to node
kubectl taint node <n> key=value:NoSchedule  # add taint
```

## configmaps & secrets

```bash
kubectl get configmaps                       # list configmaps
kubectl get cm <n> -o yaml                # view configmap
kubectl create configmap <n> --from-file=./config.conf   # from file
kubectl create configmap <n> --from-literal=key=value    # inline
kubectl delete configmap <n>              # delete

kubectl get secrets                          # list secrets
kubectl get secret <n> -o yaml            # view secret (base64 encoded)
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
kubectl delete pod,svc --all -n <ns>        # delete all pods and services in ns

# get / describe (works for any resource type)
kubectl get <resource>                       # list
kubectl get <resource> <n>                # specific
kubectl get <resource> -l key=value         # filter by label
kubectl describe <resource> <n>           # details + events

# short resource names
# po=pods, deploy=deployments, svc=services, cm=configmaps
# ns=namespaces, no=nodes, pv=persistentvolumes, pvc=persistentvolumeclaims
# ing=ingresses, sa=serviceaccounts, rs=replicasets, sts=statefulsets, ds=daemonsets

kubectl get po,svc,deploy -n <ns>           # multiple resource types at once
kubectl get all -n <ns>                      # common resources (not truly all)
```

## labels & selectors

```bash
kubectl label pod <n> env=prod            # add label
kubectl label pod <n> env-                # remove label
kubectl get pods -l env=prod                 # filter by label
kubectl get pods -l 'env in (prod,staging)' # set-based selector
kubectl get pods --show-labels               # show all labels
kubectl annotate pod <n> description="..."  # add annotation
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
kubectl get sts <n> -o yaml               # view spec
kubectl scale sts <n> --replicas=3        # scale statefulset
kubectl rollout restart sts/<n>           # rolling restart

kubectl get daemonsets                       # list daemonsets (one pod per node)
kubectl describe ds <n>                   # daemonset details
```

## persistent volumes

```bash
kubectl get pv                               # list persistent volumes (cluster-wide)
kubectl get pvc                              # list persistent volume claims (ns-scoped)
kubectl describe pv <n>                   # PV details
kubectl describe pvc <n>                  # PVC details
kubectl delete pvc <n>                    # delete PVC
```

## rbac

```bash
kubectl get roles -n <ns>                    # list roles
kubectl get rolebindings -n <ns>             # list role bindings
kubectl get clusterroles                     # cluster-wide roles
kubectl get clusterrolebindings              # cluster-wide bindings
kubectl describe role <n> -n <ns>         # role details
kubectl auth can-i <verb> <resource>         # check own permissions
kubectl auth can-i get pods --as=user@example.com  # check as another user
kubectl auth can-i --list                    # list all permissions
```

## debugging & troubleshooting

```bash
# why is a pod not starting?
kubectl describe pod <n>                  # check Events section
kubectl logs <n> --previous              # logs from crashed container

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
