## Egress Gateway

The egress gateway feature routes all IPv4 and IPv6 connections originating from pods and destined to specific cluster-external CIDRs through particular nodes, called "gateway nodes".

When the egress gateway feature is enabled and egress gateway policies are in place, matching packets that leave the cluster are masqueraded with selected, predictable IPs associated with the gateway nodes.

As an example, this feature can be used in combination with legacy firewalls to allow traffic to legacy infrastructure only from specific pods within a given namespace.

## Prerequisites

The enablement of the egress gateway feature requires both:

1. BPF masquerading
2. kube-proxy replacement

## Delay for Enforcement of Egress Policies on New Pods

When new pods are started, there is a delay before egress gateway policies are applied for those pods. That means traffic from those pods may leave the cluster with a source IP address (pod IP or node IP) that doesn't match the egress gateway IP. That egressing traffic will also not be redirected through the gateway node.

## Incompatibility with Other Features

Egress gateway is not compatible with the following:

- Identity allocation mode kvstore
- Cluster Mesh feature
- CiliumEndpointSlice feature

## Installation

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium --version 1.19.1 \
   --namespace kube-system \
   --reuse-values \
   --set egressGateway.enabled=true \
   --set bpf.masquerade=true \
   --set kubeProxyReplacement=true
```

## Selecting and Configuring the Gateway Node

The node that should act as gateway node for a given policy can be configured with the egressGateway field.

```yaml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
```

**Important notes:**

- If multiple nodes match the given set of labels, the first node in lexical ordering based on their name will be selected.
- If there is no match for the given set of labels, Cilium drops the traffic that matches the destination CIDR(s).

The IP address that should be used to SNAT traffic must also be configured. There are 3 different ways this can be achieved:

### 1. Interface

```yaml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
  interface: ethX
```

### 2. Egress IP

```yaml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
  egressIP: a.b.c.d
```

### 3. Default Route Interface

By omitting both egressIP and interface properties, the agent will use the first IPv4 and IPv6 addresses assigned to the interface for the default route:

```yaml
egressGateway:
  nodeSelector:
    matchLabels:
      testLabel: testVal
```

**Important notes:**

- The egressIP and interface properties cannot both be specified in the egressGateway spec.
- After Cilium has selected the Egress IP for an Egress Gateway policy (or failed to do so), it does not automatically respond to a change in the gateway node's network configuration (for example if an IP address is added or deleted). You can force a fresh selection by re-applying the Egress Gateway policy.

## Example Configuration

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-sample
spec:
  # Specify which pods should be subject to the current policy.
  # Multiple pod selectors can be specified.
  selectors:
  - podSelector:
      matchLabels:
        org: empire
        class: mediabot
        # The following label selects default namespace
        io.kubernetes.pod.namespace: default
    nodeSelector: # optional, if not specified the policy applies to all nodes
      matchLabels:
        node.kubernetes.io/name: node1 # only traffic from this node will be SNATed
  # Specify which destination CIDR(s) this policy applies to.
  # Multiple CIDRs can be specified.
  destinationCIDRs:
  - "0.0.0.0/0"
  - "::/0"

  # Configure the gateway node.
  egressGateway:
    # Specify which node should act as gateway for this policy.
    nodeSelector:
      matchLabels:
        node.kubernetes.io/name: node2

    # Specify the IP address used to SNAT traffic matched by the policy.
    # It must exist as an IP associated with a network interface on the instance.
    egressIP: 10.168.60.100

    # Alternatively it's possible to specify the interface to be used for egress traffic.
    # In this case the first IPv4 and IPv6 addresses assigned to that interface will be used
    # as egress IP.
    # interface: enp0s8
```

## Testing

### Verify Cluster Nodes

```bash
kubectl get nodes -o wide
NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
kube-master1   Ready    control-plane   38h   v1.35.1   168.119.62.57     <none>        Ubuntu 24.04.4 LTS   6.8.0-90-generic   containerd://1.7.28
kube-worker1   Ready    worker          38h   v1.35.1   78.47.227.138     <none>        Ubuntu 24.04.4 LTS   6.8.0-90-generic   containerd://1.7.28
kube-worker2   Ready    worker          38h   v1.35.1   138.199.225.213   <none>        Ubuntu 24.04.4 LTS   6.8.0-90-generic   containerd://1.7.28
```

### Deploy Test Client

```bash
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/kubernetes-dns/dns-sw-app.yaml
```

Verify pod is running:

```bash
kubectl get pods -o wide
NAME       READY   STATUS    RESTARTS   AGE   IP          NODE           NOMINATED NODE   READINESS GATES
mediabot   1/1     Running   0          49s   10.0.2.66   kube-worker2   <none>           <none>
```

### Test Without Policy

```bash
kubectl exec mediabot -- curl http://138.199.218.25:80
```

Check access log:

```bash
tail /var/log/nginx/access.log | head
138.199.225.213 - - [27/Feb/2026:12:16:52 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.88.1
```

Since the client pod is running on the node `138.199.225.213 (kube-worker2)`, it is expected that without any Cilium egress gateway policy in place, traffic will leave the cluster with the IP of the node.

### Apply Egress Gateway Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: egress-sample
spec:
  selectors:
  - podSelector:
      matchLabels:
        org: empire
        class: mediabot
        io.kubernetes.pod.namespace: default
  destinationCIDRs:
  - 138.199.218.25/32
  egressGateway:
    nodeSelector:
      matchLabels:
        egress-node: "true"
    # IP used to masquerade traffic leaving the cluster
    # egressIP: "192.168.60.100"
  egressGateways:
  # It's possible to specify multiple egress gateways. In this case the source
  # endpoints will egress traffic through one of the gateways listed below.
  - nodeSelector:
      matchLabels:
        egress-node: "true"
```

Label a node as the egress gateway:

```bash
kubectl label nodes kube-worker1 egress-node=true
```

### Test With Policy

```bash
kubectl exec mediabot -- curl http://138.199.218.25:80
```

Check access log:

```bash
tail /var/log/nginx/access.log | head
78.47.227.138 - - [27/Feb/2026:12:27:43 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.88.1
```

The IP address `78.47.227.138` is the node with label `egress-node=true`, confirming the policy is working correctly.

## Troubleshooting

View egress gateway rules:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg bpf egress list
Source IP   Destination CIDR    Egress IP   Gateway IP
10.0.2.66   138.199.218.25/32   0.0.0.0     78.47.227.138
```

Verify pod state:

```bash
kubectl get pods -o wide
NAME       READY   STATUS    RESTARTS   AGE   IP          NODE           NOMINATED NODE   READINESS GATES
mediabot   1/1     Running   0          28m   10.0.2.66   kube-worker2   <none>           <none>
```
