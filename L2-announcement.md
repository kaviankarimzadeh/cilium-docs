## L2 Announcements / L2 Aware LB

L2 Announcements is a feature which makes services visible and reachable on the local area network. This feature is primarily intended for on-premises deployments within networks without BGP based routing such as office or campus networks.

## What is “L2 Announcements”
Think of a service VIP (virtual IP) that should be reachable from machines on the local LAN (office / lab networks) without a BGP router.


With L2 Announcements, a single Kubernetes service IP (an ExternalIP or a LoadBalancer IP) is announced on the L2 network by one of your cluster nodes using ARP (IPv4) / NDP (IPv6). That node answers ARP/NDP for that IP with its MAC and then forwards (load-balances) incoming traffic into the cluster. This makes the service reachable from the LAN just like any other host on the LAN.


Why this is useful vs NodePort:
Each service can have its own IP (no port juggling). If the node that was announcing the IP fails, the VIP moves to another node (lease failover) so the IP keeps working.

The advantage of this feature over NodePort services is that each service can use a unique IP so multiple services can use the same port numbers.

When using NodePorts, it is up to the client to decide to which host to send traffic, and if a node goes down, the IP+Port combo becomes unusable. With L2 announcements the service VIP simply migrates to another node and will continue to work.

## How it works

* ARP/NDP reply: Nodes respond to ARP (IPv4) or NDP (IPv6) queries for the announced IPs.

* Leader election / leases: For each announced service, Cilium uses a Kubernetes Lease (coordination API) so only one node replies to ARP/NDP for that IP at a time. The lease holder sends a gratuitous ARP when it becomes leader so LAN hosts update their ARP tables.

* No pre-cluster L3 load balancing: Since only one node answers ARP for the IP, there’s no L2-level load-sharing BEFORE traffic reaches the cluster — load balancing happens after the packet arrives at the node (Cilium does service LB)


## Limitations and When Not to Use

1. **Per-packet L2 distribution:** Not suitable if you need per-packet L2 distribution before cluster — a single node will respond for the IP, so traffic concentration can occur.
2. **externalTrafficPolicy incompatibility:** Incompatible with externalTrafficPolicy: Local (can cause traffic drops).
3. **kube-proxy replacement required:** Needs kube-proxy replacement (Cilium must be running in kube-proxy replacement mode).

## Configuration

```bash
helm upgrade cilium cilium/cilium --version 1.19.1 \
   --namespace kube-system \
   --reuse-values \
   --set l2announcements.enabled=true \
   --set k8sClientRateLimit.qps=40 \
   --set k8sClientRateLimit.burst=80 \
   --set kubeProxyReplacement=true \
   --set k8sServiceHost=${API_SERVER_IP} \
   --set k8sServicePort=${API_SERVER_PORT}
```

* If you want externalIPs announced, set externalIPs.enable=true in Helm

* Sizing the client rate limit (k8sClientRateLimit.qps and k8sClientRateLimit.burst) is important when using this feature due to increased API usage

## verify

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg config --all | grep EnableL2Announcements
```

## Prerequisites

1. Kube Proxy replacement mode must be enabled
2. All devices on which L2 Aware LB will be announced should be enabled and included in the --devices flag or devices Helm option if explicitly set


## L2 Announcement Policies

Policies provide fine-grained control over which services should be announced, where, and how. This is an example policy using all optional fields:

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy1
spec:
  serviceSelector:
    matchLabels:
      color: blue
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  interfaces:
  - ^eth[0-9]+
  externalIPs: true
  loadBalancerIPs: true
```

## Example Configuration
```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: example-l2-policy
spec:
  serviceSelector: {}
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  interfaces:
    - enp7s0
  externalIPs: true
  loadBalancerIPs: true
```

⚠️ **Note:** This configuration may not work on all cloud providers (e.g., Hetzner)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: productpage-lb
  namespace: default
spec:
  type: LoadBalancer
  loadBalancerClass: io.cilium/l2-announcer
  loadBalancerIP: 10.0.1.200
  selector:
    app: productpage
  ports:
  - name: http
    protocol: TCP
    port: 9080
    targetPort: 9080
```

⚠️ **Note:** Using externalIPs may also face similar limitations on certain cloud providers
```yaml
apiVersion: v1
kind: Service
metadata:
  name: productpage-lb
  namespace: default
spec:
  externalIPs:
    - 10.0.1.200
  selector:
    app: productpage
  ports:
  - name: http
    protocol: TCP
    port: 9080
    targetPort: 9080
```

### Service Selector
The service selector is a label selector that determines which services are selected by this policy. If no service selector is provided, all services are selected by the policy

### Node Selector
The node selector field is a label selector which determines which nodes are candidates to announce the services from.

### Interfaces
The interfaces field is a list of regular expressions (golang syntax) that determine over which network interfaces the selected services will be announced. This field is optional, if not specified all interfaces will be used.

### IP Types
The externalIPs and loadBalancerIPs fields determine what sort of IPs are announced. They are both set to `false` by default, so `a functional policy should always have one or both set to true`.

1. If `externalIPs` is true all IPs in .spec.externalIPs field are announced. These IPs are managed by service authors.
2. If `loadBalancerIPs` is true all IPs in the service’s .status.loadbalancer.ingress field are announced.

