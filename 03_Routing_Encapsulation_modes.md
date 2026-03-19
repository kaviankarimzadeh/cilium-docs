### 📦 VXLAN vs Geneve (in Cilium encapsulation)


🔹 1. VXLAN (older, simpler, widely used)

🧠 What it is
Standardized encapsulation protocol (RFC 7348)
Uses:
* UDP port 8472
* Adds a fixed header (~50 bytes total overhead)


⚙️ Key characteristics
✔ Simple & stable

* Fixed header format
*  Very widely supported (kernel, hardware, cloud)

✔ Fast
* Minimal processing
* Efficient in most environments

❌ Limited metadata
Only supports:
* VNI (VXLAN Network Identifier)
* Cannot easily carry extra info

📌 In Cilium
* Historically the default encapsulation
* Works everywhere with minimal surprises


🔹 2. Geneve (newer, flexible, Cilium-friendly)

🧠 What it is
Modern encapsulation protocol (RFC 8926)
* Uses: UDP port 6081
* Designed to replace VXLAN

⚙️ Key characteristics
✔ Extensible header
* Has TLV (Type-Length-Value) options
* Can carry arbitrary metadata
👉 This is the big deal

🚀 Why Cilium likes Geneve
Cilium can embed:
* Security identity
* Policy info

👉 This enables:
* Faster policy enforcement
* Less need for lookups in datapath

⚠️ Downsides
❌ Slightly more overhead
* More flexible header = slightly heavier
❌ Less hardware offload support
* Some NICs optimize VXLAN better than Geneve


⚔️ Side-by-side comparison
| Feature            | VXLAN            | Geneve           |
| ------------------ | ---------------- | ---------------- |
| Standard           | Older (RFC 7348) | Newer (RFC 8926) |
| UDP Port           | 8472             | 6081             |
| Header             | Fixed            | Flexible (TLV)   |
| Metadata support   | ❌ Very limited   | ✅ Rich           |
| Performance        | Slightly faster  | Slightly heavier |
| Hardware offload   | ✅ Better         | ⚠️ Less common   |
| Cilium integration | Good             | **Best**         |


---
⚡ When to choose what:

✅ Choose VXLAN if:
You rely on:
* NIC offloading
* legacy infra
* You want maximum compatibility

✅ Choose Geneve if:
You want:
* best Cilium performance
* dvanced policy scaling
* You’re fully software-defined (no hardware limits)

---

⚙️ How to enable VXLAN vs Geneve in Cilium

🔹 Enable VXLAN (default)
```bash
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan
```


🔹 Enable Geneve
```
  --set routingMode=tunnel \
  --set tunnelProtocol=geneve
```

### 🔹 Verify what is running

```
kubectl -n kube-system exec ds/cilium -- cilium status | grep Tunnel
```

Expected:
* VXLAN → Tunnel: vxlan
* Geneve → Tunnel: geneve


#### Check UDP ports on nodes
```
ss -lunp | grep -E "8472|6081"
```

VXLAN → UDP 8472
Geneve → UDP 6081

#### tcpdump (best proof)

```bash
tcpdump -i any udp port 8472   # VXLAN
tcpdump -i any udp port 6081   # Geneve
```

---

🧪 How to benchmark

You want to measure:
* Throughput
* Latency
* CPU overhead

#### 🔹 Step 1: Create test pods on different nodes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: iperf-server
spec:
  nodeName: <node-1>
  containers:
  - name: iperf
    image: networkstatic/iperf3
    command: ["iperf3", "-s"]
---
apiVersion: v1
kind: Pod
metadata:
  name: iperf-client
spec:
  nodeName: <node-2>
  containers:
  - name: iperf
    image: networkstatic/iperf3
    command: ["sleep", "3600"]
```


🔹 Step 2: Run throughput test

```bash
kubectl exec -it iperf-client -- iperf3 -c <server-pod-ip> -t 30
```

Measure:
* Gbits/sec
* Retransmits

🔹 Step 3: Run latency test

```bash
kubectl exec -it iperf-client -- ping <server-pod-ip>
```

Compare:
* avg latency
* jitter


🔹 Step 4: Measure CPU overhead

```bash
kubectl top pods -n kube-system | grep cilium
```

📊 3. What differences you should expect

👉 In reality:
| Metric            | VXLAN              | Geneve             |
| ----------------- | ------------------ | ------------------ |
| Throughput        | 🟢 Slightly higher | 🟡 Slightly lower  |
| Latency           | 🟢 Slightly lower  | 🟡 Slightly higher |
| CPU               | 🟢 Lower           | 🔴 Higher          |
| Cilium efficiency | 🟡 Good            | 🟢 Better          |


🚀 Advanced: Real-world comparison (better test)

Run parallel:
```bash
iperf3 -c <ip> -t 60 -P 4
```

