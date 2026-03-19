```bash
kind create cluster --config kind.config
kubectl cluster-info --context kind-cilium-cluster
kubectl get pods -A
```

