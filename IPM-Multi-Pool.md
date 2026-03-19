# Multi-Pool

The Multi-Pool IPAM mode supports allocating PodCIDRs from multiple different IPAM pools, depending on workload annotations and node labels defined by the user.


How it works (conceptually)

1. You define multiple CIDR pools using Cilium CRDs
2. Each pool can be:

IPv4 / IPv6
Routed differently
Used by specific namespaces, nodes, or workloads

3. Pods are assigned IPs from the matching pool, not a global one
👉 Kubernetes itself (Kubernetes) is not aware of these pools — this is fully handled by Cilium.



Points regarding requirements:

Enable automatic node CIDR allocation (Recommended)
Kubernetes has the capability to automatically allocate and assign a per node IP allocation CIDR. Cilium automatically uses this feature if enabled. This is the easiest method to handle IP allocation in a Kubernetes cluster. To enable this feature, simply add the following flag when starting kube-controller-manager:

--allocate-node-cidrs 