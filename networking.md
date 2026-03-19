## Masquerading

IPv4 addresses used for pods are typically allocated from RFC1918 private address blocks and thus, not publicly routable. Cilium will automatically masquerade the source IP address of all traffic that is leaving the cluster to the IPv4 address of the node as the node’s IP address is already routable on the network.

## Implementation Modes

# eBPF-based
The eBPF-based implementation is the most efficient implementation. It can be enabled with the bpf.masquerade=true helm option.

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Masquerading
Masquerading:            IPTables [IPv4: Enabled, IPv6: Disabled]
```

The eBPF-based masquerading can masquerade packets of the following L4 protocols:

TCP
UDP
ICMP

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


# iptables-based

This is the legacy implementation that will work on all kernel versions.

The default behavior will masquerade all traffic leaving on a non-Cilium network device. This typically leads to the correct behavior. In order to limit the network interface on which masquerading should be performed, the option egress-masquerade-interfaces: eth0 can be used.


## Default Ingress Allow from Local Host

Kubernetes has functionality to indicate to users the current health of their applications via Liveness Probes and Readiness Probes. In order for kubelet to run these health checks for each pod, by default, Cilium will always allow all ingress traffic from the local host to each pod.



## Enable automatic node CIDR allocation (Recommended)

Kubernetes has the capability to automatically allocate and assign a per node IP allocation CIDR. Cilium automatically uses this feature if enabled. This is the easiest method to handle IP allocation in a Kubernetes cluster. To enable this feature, simply add the following flag when starting kube-controller-manager:

```bash
--allocate-node-cidrs
```


## routing modes

🚦 1. Encapsulation (Tunnel) Mode — the Default
What it is:
Every packet between pods on different nodes is wrapped (encapsulated) in another header and sent through a tunnel (VXLAN or Geneve). Think of it like putting your message in an envelope before delivery.
How it works:
Nodes form a mesh of tunnels.
Underlying network just sees traffic between node IPs; it doesn’t care about individual Pod addresses.
Pros (simple and safe):
🧩 Easy to set up — no special routing on layer 3 needed.
🌍 Works even if your physical network doesn’t know about pods’ IPs.
🔄 New nodes join automatically.
Cons (overhead):
📦 Extra tunnel headers shrink usable packet size (MTU) and can slightly reduce throughput.
📌 This is the default because it “just works” on almost every infrastructure.


🚀 2. Native Routing (Direct Mode)
What it is:
Instead of tunnels, Cilium uses the host’s routing tables to send packets directly via the underlying network — no wrapping.
How it works:
Pod IP routes are added to the Linux routing table.
Traffic is forwarded directly if the network knows how to reach pods.
Pros (performance):
⚡ Lower latency — no encapsulation overhead.
📉 Better throughput (larger MTU).
🐝 Works well in flat or routable networks (cloud VPCs, well-configured on-prem).
Cons (harder setup):
🛠 Requires that your network can route all Pod IPs.
Nodes must learn routing to each pod.
Either via kernel routes (auto-direct-node-routes) or a routing protocol like BGP.


🧠 Which Mode Is Best for Kubernetes?
There isn’t one “magic best mode” — it depends on your environment:
✅ Use Encapsulation (Default) if:
You want zero extra network setup.
Your cluster runs on a network that does not support direct routing of Pod CIDRs (e.g., many on-prem or multi-datacenter setups).
Simplicity and reliability matter.
🤝 This is perfect for many Kubernetes clusters because it avoids requiring advanced routing.
🌐 Use Native Routing if:
You run in a cloud (like AWS, GKE, Azure) or your network supports routing all Pod IPs.
You want higher performance (lower overhead, larger MTU).
You can manage network routing (BGP, static routes, etc.).
📌 For production Kubernetes where performance matters and the underlay network is capable, Native Routing often gives better throughput. But it does require more networking knowledge.

