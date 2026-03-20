## Maglev

Maglev is a consistent hashing–based load-balancing algorithm originally designed by Google for the Cilium kube-proxy replacement. For more details, see the [Cilium documentation](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#maglev-consistent-hashing).

## How to Enable

```bash
head -c12 /dev/urandom | base64 -w0
```

```bash
cilium upgrade --set loadBalancer.algorithm=maglev --set maglev.hashSeed=cQbV704rk6lMw92b
```

## What is Maglev?

Maglev is a consistent hashing–based load-balancing algorithm originally designed by Google.

## Goal

Distribute traffic evenly while minimizing reshuffling when backends change.

In Kubernetes terms:

Service → multiple Pods (backends)

When Pods scale up/down → traffic shouldn't completely reshuffle

### The Core Problem It Solves

#### Normal Load Balancing (Round-Robin / Random)

If you have:
```
Service A → Pods [P1, P2, P3]
```

Clients get distributed evenly.

Now you scale:
```
Service A → Pods [P1, P2, P3, P4]
```

With round-robin or random:

```
👉 Almost all clients may get reassigned
👉 Cache locality breaks
👉 Session stickiness breaks
👉 More reconnections
👉 Higher latency spike
```

This is very bad for:
```
Stateful services
In-memory cache systems
gRPC persistent connections
Gaming backends
AI inference pods with warm cache
```

### How Maglev Works (Simplified)

Maglev creates a large lookup table.

Example (simplified):
Table size = 13 slots

Pods:
```
P1
P2
P3
```

Maglev builds a permutation for each backend and fills the table in a deterministic way.

Example lookup table:

```
Index:  0 1 2 3 4 5 6 7 8 9 10 11 12
Value: P1 P2 P3 P1 P2 P3 P1 P2 P3 P1 P2 P3 P1
```

When a packet arrives:

```
hash(client_ip, client_port, dst_ip, dst_port) % table_size
```

That index selects the backend.

### Why It's Special

When a new pod P4 is added:
Maglev rebuilds the table, but:

```
👉 Only a small percentage of slots change
👉 Most client → backend mappings stay the same
```

This is the key benefit.

### Real Example With Numbers

Assume:
```
10,000 active client connections
3 pods → scale to 4 pods
```

With round-robin:
```
~75% of clients get reassigned
```

With Maglev:
```
~25% get reassigned (only proportional to new capacity)
```

That means:
```
Fewer TCP resets
Less cache invalidation
Smoother autoscaling
```

### In Cilium Specifically

When kube-proxy replacement is enabled:

**Cilium implements Maglev in eBPF in kernel space.**

That means:

* No iptables rules
* No userspace LB
* No IPVS table overhead
* O(1) lookup time
* Extremely fast

And it's applied to:
* ClusterIP
* NodePort
* LoadBalancer services

### Why It's Better Than IPVS Consistent Hashing

| Feature                     | IPVS    | Cilium Maglev |
| --------------------------- | ------- | ------------- |
| True consistent hashing     | Limited | Yes           |
| Minimal reshuffle guarantee | ❌       | ✅             |
| Kernel eBPF implementation  | ❌       | ✅             |
| Works with DSR              | ❌       | ✅             |
| Per-service configurable    | Limited | Yes           |

### Trade-offs / Limitations

It's not magic.

**Memory Usage**

Maglev uses a lookup table with a default size often around 65,537 entries. This uses more memory than simple round-robin load balancing, but the overhead is usually negligible in modern nodes.

### When Should You Use It?

**Use Maglev if:**
- You have autoscaling workloads
- You care about latency spikes during scaling events
- You have cache-heavy workloads
- You run high-QPS APIs
- You run multi-zone clusters
- You run large production systems

**Not necessary if:**
- Small cluster with simple needs
- Very simple stateless services
- No scaling activity expected

## Interaction with HPA Scaling

### What Happens During HPA Scaling?

Suppose you have:
```
Service A → 3 Pods
HPA min=3 max=10
```

Traffic increases → HPA scales:

```
3 → 6 Pods
```

Traffic decreases → HPA scales:

```
6 → 4 Pods
```

Every scale event changes the backend set.

Now the question:

What happens to existing connections and traffic distribution?
That's where load-balancing algorithm matters.

### Without Consistent Hashing (Round-Robin / Random)

Clients distributed evenly.

Most traditional LBs will:
* Recalculate distribution
* Redistribute traffic almost randomly
* Many clients switch pods

Result:
* Cache locality destroyed
* TCP connections may reset
* gRPC streams break
* Cold start spikes
* Latency spikes
* CPU spikes
* Autoscaling feedback loop instability

This creates a scaling shockwave.

### With Maglev During Scale-Up

Maglev builds lookup table.

Maglev rebuilds table after scale happens.

But here's the key:

👉 Only ~50% of slots change (proportional to new capacity).
👉 ~50% of existing connections still map to same backend.

More generally:

If you go from N → N+k:
Only about k/(N+k) of flows get remapped.

Example:
3 → 6 pods
Remap ≈ 3/6 = 50%

10 → 11 pods
Remap ≈ 1/11 ≈ 9%

That's much smaller disruption.

### Why This Matters for HPA Specifically

HPA reacts to CPU/memory/metrics.

If load balancing reshuffles too aggressively:

1. Some pods suddenly lose traffic
2. Some pods suddenly get too much traffic
3. Metrics fluctuate wildly
4. HPA may overreact
5. Oscillation happens

Maglev reduces that metric instability.

### Important: What Maglev Does NOT Solve

Maglev does NOT:

* Drain connections automatically on scale-down
* Guarantee zero reshuffle
* Replace proper readiness probes
* Replace connection draining logic

You still need:

* Proper readiness probes
* Graceful termination (preStop hooks)
* terminationGracePeriodSeconds
* PodDisruptionBudget

Maglev just reduces disruption amplitude.

### Summary: HPA + Maglev

| Situation               | Without Maglev   | With Maglev                    |
| ----------------------- | ---------------- | ------------------------------ |
| Scale up                | Large reshuffle  | Minimal proportional reshuffle |
| Scale down              | Wide instability | Only removed pods affected     |
| Cache hit rate          | Drops heavily    | Mostly preserved               |
| CPU stability           | Spiky            | Smooth                         |
| gRPC stability          | Reconnect storms | Stable                         |
| Autoscaling oscillation | Higher           | Lower                          |
