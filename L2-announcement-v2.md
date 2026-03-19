## Refrences

### LoadBalancer IP Address Management (LB IPAM)
https://docs.cilium.io/en/latest/network/lb-ipam/ 

`LB IPAM → assigns the IP`

### L2 Announcements / L2 Aware LB
https://docs.cilium.io/en/latest/network/l2-announcements/

`L2 Announcements → makes that IP reachable on the network`

The two features are related but solve different parts of the LoadBalancer problem in Cilium. Think of them as two layers of the same system.


1. `LB IPAM (LoadBalancer IP Address Management)`
This feature allocates IP addresses to Kubernetes Service `type=LoadBalancer`.
In `cloud environments (AWS/GCP/Azure)`, the cloud provider **automatically** assigns a LoadBalancer IP.

In `bare-metal or private cloud`, that does not exist, so `Cilium` provides LB IPAM.

What it does
You create an `IP pool`
Cilium automatically assigns an IP from that pool to your service

### example

```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: public-pool
spec:
  blocks:
  - cidr: 192.168.10.0/24
```

```yaml
apiVersion: v1
kind: Service
spec:
  type: LoadBalancer
```

Result:
`Service -> gets IP 192.168.10.15`

👉 **LB IPAM only assigns the IP.**

It does `NOT` make the IP reachable externally.
Something else must advertise that IP.


2. `L2 Announcements`
This feature announces that IP to the local network using ARP/NDP so external machines can reach it.
It works similar to `MetalLB L2` mode.

### What it does
One node becomes the announcer

That node answers ARP requests for the service IP

Traffic enters the cluster through that node


```bash
Client -> ARP: who has 192.168.10.15?
NodeA -> replies with its MAC
Client -> sends traffic to NodeA
NodeA -> forwards to service pods
```
### Characteristics:
Works without BGP

Works only in the same L2 network

One node handles entry traffic for that service

Failover occurs if that node dies


## How they work together
Typical bare-metal setup:

```bash
Service (LoadBalancer)
       │
       ▼
LB IPAM
(assigns IP)
       │
       ▼
L2 Announcements
(advertises IP with ARP)
       │
       ▼
External clients can reach the service
```

### Example flow:
```bash
1. Service created
2. LB IPAM → assigns 192.168.10.15
3. L2 Announcer → advertises ARP for 192.168.10.15
4. Client connects
```


## configuration

#### Enable Required Features in Cilium
```yaml
l2announcements:
  enabled: true

externalIPs:
  enabled: true

k8sClientRateLimit:
  qps: 50
  burst: 100

ipam:
  mode: kubernetes

enableLBIPAM: true
```

#### Configure LB IPAM Pool

```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: production-pool
spec:
  blocks:
    - cidr: 192.168.1.200/29
```

#### Example IPs available:
```bash
192.168.1.200
192.168.1.201
192.168.1.202
192.168.1.203
192.168.1.204
192.168.1.205
```

#### Configure L2 Announcement Policy
This tells Cilium which nodes are allowed to announce the IPs.

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: announce-lb-services
spec:
  serviceSelector:        ## Only services with this label will be announced
    matchLabels:
      expose: "true"

  nodeSelector:           ## Which nodes can announce
    matchLabels:
      kubernetes.io/os: linux

  interfaces:
    - eth0                ## Network interface used for ARP

  externalIPs: true
  loadBalancerIPs: true   ## Announce LoadBalancer IPs
```


#### Example Application Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    expose: "true"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

#### checking

```bash
kubectl get svc

NAME    TYPE           CLUSTER-IP     EXTERNAL-IP
nginx   LoadBalancer   10.96.12.10    192.168.1.201
```

#### Verify L2 Announcement

```bash
arp -a | grep 192.168.1.201`
```
You will see the MAC of the announcing node.

#### Failover Behavior

If the announcing node dies:
```bash
NodeA (announcer) dies
        ↓
Cilium elects NodeB
        ↓
NodeB starts answering ARP
```

**Failover usually takes a few seconds.**


## KodeKloud Examples (Kind cluster)

```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    - start: "172.19.0.240" ## same subnet as nodes in Kind cluster!
      stop: "172.19.0.250"
```

```bash
kubectl get svc
```

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: announce-lb-services
spec:
  serviceSelector:
    matchLabels:
      app: "myapp"

  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/control-plane
      operator: DoesNotExist ## means all nodes except the control plane node

  interfaces:
    - ^eth[0-9]+                ## Network interface used for ARP

  externalIPs: true
  loadBalancerIPs: true   ## Announce LoadBalancer IPs
```


```bash
kubectl get lease
```

