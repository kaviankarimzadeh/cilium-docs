## Masquerading

IPv4 addresses used for pods are typically allocated from RFC1918 private address blocks and thus, not publicly routable. Cilium will automatically masquerade the source IP address of all traffic that is leaving the cluster to the IPv4 address of the node as the node’s IP address is already routable on the network.

### check via CLI

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Masquerading
Masquerading:            IPTables [IPv4: Enabled, IPv6: Disabled]
```

---
### Implementation Modes

#### iptables-based

This is the legacy implementation that will work on all kernel versions.

The default behavior will masquerade all traffic leaving on a non-Cilium network device. This typically leads to the correct behavior. In order to limit the network interface on which masquerading should be performed, the option `egress-masquerade-interfaces: eth0` can be used.



#### eBPF-based
The eBPF-based implementation is the most efficient implementation. It can be enabled with the `bpf.masquerade=true` helm option.

The eBPF-based masquerading can masquerade packets of the following L4 protocols:

- TCP
- UDP
- ICMP

By default, all packets from a pod destined to an IP address outside of the ipv4-native-routing-cidr range are masqueraded, except for packets destined to other cluster nodes (as with ipv6-native-routing-cidr for IPv6).


When eBPF-masquerading is enabled, traffic from pods to the External IP of cluster nodes will also not be masqueraded. The eBPF implementation differs from the iptables-based masquerading on that aspect


The eBPF-based ip-masq-agent supports the nonMasqueradeCIDRs, masqLinkLocal, and masqLinkLocalIPv6 options set in a configuration file. A packet sent from a pod to a destination which belongs to any CIDR from the nonMasqueradeCIDRs is not going to be masqueraded. If the configuration file is empty, the agent will provision the following non-masquerade CIDRs:

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
100.64.0.0/10
192.0.0.0/24
192.0.2.0/24
192.88.99.0/24
198.18.0.0/15
198.51.100.0/24
203.0.113.0/24
240.0.0.0/4
```

## how to enable

```bash
cilium upgrade --set bpf.masquerade=true 
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --all-addresses
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Masquerading

```


## Enable automatic node CIDR allocation (Recommended)

Kubernetes has the capability to automatically allocate and assign a per node IP allocation CIDR. Cilium automatically uses this feature if enabled. This is the easiest method to handle IP allocation in a Kubernetes cluster. To enable this feature, simply add the following flag when starting `kube-controller-manager`:

```bash
--allocate-node-cidrs
```

---

## Why BPF is faster (architectural reason)
⚙️ iptables mode (default)
* Uses netfilter hooks
* Traverses rule chains (can grow large)
* Happens later in the networking stack
* Involves more context switches + packet path overhead

👉 Complexity grows with rules → performance degrades at scale


⚙️ eBPF mode
* Runs directly in kernel datapath
* Attached to interfaces (TC/XDP level)
* No rule-chain traversal
* Works with maps (O(1) lookups)

👉 More predictable + lower latency


## Real performance difference (practical view)

📊 BPF masquerade helps especially in:
* High PPS (packets per second)
* Many connections (microservices, service mesh)
* Large clusters (lots of rules / nodes)
* Heavy egress traffic (SNAT-heavy workloads)


✅ Final verdict

| Aspect      | iptables      | BPF masquerade      |
| ----------- | ------------- | ------------------- |
| Performance | ❌ slower      | ✅ faster            |
| Scalability | ❌ rule-based  | ✅ map-based         |
| CPU usage   | ❌ higher      | ✅ lower             |
| Maturity    | ✅ very stable | ✅ stable (IPv4)     |
| Flexibility | ✅ predictable | ⚠️ some differences |
