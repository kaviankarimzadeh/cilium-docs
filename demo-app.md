## Cilium demo app


![alt text](https://docs.cilium.io/en/stable/_images/cilium_http_gsg.png)


### Validate the Installation

```bash
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/minikube/http-sw-app.yaml
```
```bash
kubectl get pods,svc
kubectl get pods --show-labels 
```

### Check default Access
```bash
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

### destroy
```bash
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/minikube/http-sw-app.yaml
```