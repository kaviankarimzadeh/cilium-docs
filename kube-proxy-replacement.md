## kubeadm init

### Pros

- low latency
- high scalability
- more efficient load balancing
- better observability and monitoring

### migrating from kube-proxy to replacement

```bash
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
# on each node
iptables-save | grep -v KUBE | iptables-restore
```


```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
networking:
  podSubnet: {{pod_subnet}}
kubernetesVersion: {{kubernetes_version}}
controlPlaneEndpoint: "{{cp_endpoint}}"
certificatesDir: "/etc/kubernetes/pki"
proxy:
  disabled: true
```


## Installation

```bash
cilium install --version 1.19.1 --set kubeProxyReplacement=true --set k8sServiceHost=168.119.62.57 --set k8sServicePort=6443 --namespace kube-system
```


```bash
cilium status --wait
```

```bash
kubectl -n kube-system get pods -l k8s-app=cilium

kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep KubeProxyReplacement

kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose

kubectl -n kube-system exec ds/cilium -- cilium-dbg service list
```

```bash
iptables-save | grep KUBE-SVC
```

When no output is returned, it confirms that kube-proxy has been successfully replaced by Cilium.