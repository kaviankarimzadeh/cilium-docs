## Quick Start Installation

Install Cilium using the CLI:

```bash
cilium install --version 1.19.1
```

Wait for the installation to complete and verify status:

```bash
cilium status --wait
```

Run connectivity tests:

```bash
cilium connectivity test
```

## Verification

Check Cilium pods:

```bash
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

List endpoints:

```bash
kubectl -n kube-system exec cilium-zg4jc -- cilium-dbg endpoint list
```

Check status details:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --all-addresses
```


## Helm Installation Methods

Cilium can be installed using Helm in two ways:

1. OCI Registry (Recommended) — Install directly from OCI registries without adding a Helm repository
2. Traditional Repository — Use the classic https://helm.cilium.io/ repository


## Restart unmanaged Pods

If you did not create a cluster with the nodes tainted with the taint node.cilium.io/agent-not-ready, then unmanaged pods need to be restarted manually. Restart all already running pods which are not running in host-networking mode to ensure that Cilium starts managing them.

```
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```
