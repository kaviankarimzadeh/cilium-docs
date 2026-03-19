# Cluster (default)

📌 How it works

Cilium manages IP allocation itself at cluster level. `Cilium operator` will manage the per-node PodCIDRs via the v2.CiliumNode resource
IPs are allocated dynamically from a shared cluster-wide pool.
Nodes don’t need predefined PodCIDRs from Kubernetes.
Allocation is coordinated via Cilium CRDs.

✅ Pros
✔ Much more flexible

No rigid per-node CIDR.
Nodes consume IPs as needed.
Better for auto-scaling clusters.
it does not depend on Kubernetes being configured to hand out PodCIDRs
NO kube-controller-manager flags needed

✔ More efficient IP usage

Reduces waste.
Better utilization in bursty environments.

✔ Better for large clusters

Especially useful when:
Many nodes
Frequent node churn
On-prem with limited IP space

✔ Required for some advanced Cilium features

Often used in:
Bare-metal
Multi-cluster
Advanced routing setups



❌ Cons

❌ More Cilium-dependent

IPAM logic moves from Kubernetes → Cilium.
Tighter coupling to Cilium.

❌ Slightly more operational complexity

Requires proper CRD handling.
Debugging IP allocation means understanding Cilium internals.

❌ Not always necessary

Overkill for small/simple clusters.


## How to enable

```yaml
ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: 10.0.0.0/8   # overall pool
    clusterPoolIPv4MaskSize: 24               # per-node slice
```

### Important operational rules
Never modify existing elements in c`lusterPoolIPv4PodCIDRList` — changes cause unexpected behavior. If the pool is exhausted, add a new element to the list instead. 
The minimum mask length is `/30`, with a recommended minimum of at least `/29`. 
Changing `clusterPoolIPv4MaskSize` after initial install is also not possible. 

#### Look for allocation errors

```bash
kubectl get ciliumnodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.ipam.operator-status}{"\n"}{end}'
```

## verification

```bash
cilium-dbg status --all-addresses
```
