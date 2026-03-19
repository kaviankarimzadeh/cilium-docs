## modes

🧠 Big Picture
Cilium gives you two ways for pods on different nodes to talk:

| Mode                        | Idea                                    |
| --------------------------- | --------------------------------------- |
| **Encapsulation (default)** | Wrap packet → send via tunnel           |
| **Native routing**          | Send packet directly using real routing |


1. Encapsulation
2. Native Routing

### Encapsulation (tunnel) (default)
if our nodes have physical connectivity to each other.

🔹 What actually happens
Pod A → Pod B (different node)
Cilium:
Takes the original packet
Wraps it inside another packet (UDP tunnel)
Sends it to the destination node
Destination node unwraps it and delivers to Pod B
👉 So the underlying network never sees pod IPs, only node IPs.

🔹 Key characteristics
1. Overlay network (tunnel mesh)
Every node builds tunnels to every other node
Protocols:
VXLAN (default, UDP 8472)
Geneve (UDP 6081)

2. Network requirements = minimal
Just node ↔ node connectivity is enough
No need for:
BGP
route distribution
pod CIDR awareness
👉 This is why it’s the default mode

🔹 Downsides
MTU overhead
~50 bytes lost per packet (VXLAN)
Can cause:
fragmentation
lower throughput


##### Encapsulation modes
1. VXLAN (default)
2. Geneve

##### Pros
- simplified network configuration
- works out of the box
- no requirement for underlay network changes
- work across different network topologies
- less dependency on Cloud 


##### Cons
- increased overhead
- Higher latency
- More complex debugging
- Potential for fragmentation



### Native Routing
use BGP Peering!

🔹 What actually happens
Pod A → Pod B (different node)
Cilium:
does NOT encapsulate
just hands packet to Linux routing
👉 Packet is sent as-is, like normal IP traffic

🔹 Key behavior
“Cilium delegates packets to Linux routing subsystem”
So:
Kernel routing decides where to send it
Network must understand pod IPs


🔹 Requirements (THIS is the big deal)
Your network must know how to reach Pod CIDRs


##### Pros
- Better performance
- More efficient use of MTU
- Easier debugging
- Optimized for large-scale deployments

##### Cons
- requires for underlay network changes
- complex routing configuration
- no native Cloud dependecy


⚔️ Encapsulation vs Native (Real Comparison)

| Feature                   | Encapsulation | Native Routing |
| ------------------------- | ------------- | -------------- |
| Setup                     | Easy          | Complex        |
| Network awareness of pods | ❌ No          | ✅ Yes          |
| Performance               | Lower         | Higher         |
| MTU                       | Reduced       | Full           |
| Requires BGP              | ❌             | Often ✅        |
| Default                   | ✅             | ❌              |


✅ Use Native Routing if:
You have:
BGP (very likely in your setup)

or flat L2 network

You care about:
performance

latency

Large clusters


