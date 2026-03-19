```bash
cilium install cilium ./cilium \
   --namespace kube-system \
   --set l2announcements.enabled=true \
   --set k8sClientRateLimit.qps=50 \
   --set k8sClientRateLimit.burst=100 \
   --set kubeProxyReplacement=true \
   --set k8sServiceHost=cilium-cluster-control-plane \
   --set k8sServicePort=6443
```

```bash
cilium status --wait
```

```bash
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  labels:
    app: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: hashicorp/http-echo
        args:
        - -listen=:80
        - -text="This is app1"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
  labels:
    app: "myapp"
spec:
  selector:
    app: app1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

```yaml
kubectl get svc 
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
app1-service   LoadBalancer   10.96.149.51   <pending>     80:30224/TCP   40s
```

```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    - start: "172.18.0.240"
      stop: "172.18.0.250"
```

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: announce-lb-services
spec:
  # serviceSelector:   # If no service selector is provided, all services are selected by the policy
  #   matchLabels:
  #     app: "myapp"

  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/control-plane
      operator: DoesNotExist ## means all nodes except the control plane node

  interfaces:
    - ^eth[0-9]+                ## Network interface used for ARP

  externalIPs: true
  loadBalancerIPs: true   ## Announce LoadBalancer IPs
```

```bash
docker exec -ti 643e9ab80e76 curl 172.18.0.240
"This is app1"
```


```bash
kubectl get leases -n kube-system
```
