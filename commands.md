## cilium-dbg

The Cilium debug CLI client (cilium-dbg) is a command-line tool that is installed along with the Cilium agent. It interacts with the REST API of the Cilium agent running on the same node. The debug CLI allows inspecting the state and status of the local agent. It also provides tooling to directly access the eBPF maps to validate their state.

## hubble-cli

The Hubble CLI (hubble) is a command-line tool able to connect to either the gRPC API of hubble-relay or the local server to retrieve flow events.


### Validate the Installation

```bash
cilium status --wait
```

### validate that your cluster has proper network connectivity

```bash
cilium connectivity test
```

### list of endpoints on each cilium node
```bash
kubectl -n kube-system get pods -l k8s-app=cilium
NAME           READY   STATUS    RESTARTS   AGE
cilium-5ngzd   1/1     Running   0          3m19s
```

```bash
kubectl -n kube-system exec cilium-5ngzd -- cilium-dbg endpoint list
```

### Inspecting the Policy
policy enforcement enabled
```bash
kubectl -n kube-system exec cilium-tqzvw  -- cilium-dbg endpoint list
```

### observe the L7 policy

```bash
kubectl -n kube-system exec cilium-tqzvw -- cilium-dbg policy get
```

###  monitor the HTTP requests live

```bash
kubectl exec -it -n kube-system cilium-tqzvw -- cilium-dbg monitor -v --type l7
```


### Restart unmanaged Pods

```bash
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```


### status via ds

```bash
kubectl -n kube-system exec ds/cilium -- cilium status
```


### check masquarade status
```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Masquerading
```

### check masquarade ip 
```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg bpf ipmasq list
```


### check KubeProxyReplacement
```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep KubeProxyReplacement
```

### full details
```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose
```


### service list
```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg service list
```


### verify egress 

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg bpf egress list
```

### egress stats

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg shell -- db/show nat-stats
```