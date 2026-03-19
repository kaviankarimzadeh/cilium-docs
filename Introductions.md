## What is Cilium?
At the foundation of Cilium is a new Linux kernel technology called eBPF, which enables the dynamic insertion of powerful security visibility and control logic within Linux itself. Because eBPF runs inside the Linux kernel, Cilium security policies can be applied and updated without any changes to the application code or container configuration.

## What is Hubble?
Hubble is a fully distributed networking and security observability platform. It is built on top of Cilium and eBPF to enable deep visibility into the communication and behavior of services as well as the networking infrastructure in a completely transparent manner.

## Why Cilium & Hubble?
eBPF is enabling visibility into and control over systems and applications at a granularity and efficiency that was not possible before. It does so in a completely transparent way, without requiring the application to change in any way.


## Load Balancing

**East-West Load Balancing:** Rewrites service connections at the socket level (connect()), avoiding the overhead of per-packet NAT and fully replacing kube-proxy. Used for traffic between pods within the cluster.

**North-South Load Balancing:** Supports XDP for high-throughput scenarios and layer 4 load balancing including Direct Server Return (DSR) and Maglev consistent hashing. Used for traffic ingressing/egressing from outside the cluster.