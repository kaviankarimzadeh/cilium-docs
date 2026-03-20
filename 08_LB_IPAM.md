## LoadBalancer IP Address Management (LB IPAM)
LB IPAM is a feature that allows Cilium to assign IP addresses to Services of type LoadBalancer. 

This functionality is usually left up to a cloud provider, however, when deploying in a private cloud environment, these facilities are not always available.

`LB IPAM` works in conjunction with features such as `Cilium BGP Control Plane` and `L2 Announcements / L2 Aware LB (Beta)`. 

Where `LB IPAM` is responsible for **allocation and assigning of IPs to Service objects** and

`Cilium BGP Control Plane` and `L2 Announcements / L2 Aware LB (Beta)` are responsible for **load balancing and/or advertisement of these IPs.**

Use `Cilium BGP Control Plane` to advertise the IP addresses assigned by LB IPAM **over BGP** and `L2 Announcements / L2 Aware LB (Beta)` to advertise them **locally**.

`LB IPAM` is always **enabled** but dormant. The controller is awoken **when the first IP Pool is added to the cluster.**

**so it's already enabled and we just need to add IPPool.**

---

### Pools

LB IPAM has the notion of IP Pools which the administrator can create to tell Cilium which IP ranges can be used to allocate IPs from.


```yaml
apiVersion: "cilium.io/v2"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "blue-pool"
spec:
  blocks:
  - cidr: "10.0.10.0/24"
  ## and/or
  - start: "20.0.20.100"
    stop: "20.0.20.200"
```


### Service Selectors
IP Pools have an optional .spec.serviceSelector field which allows administrators to limit which services can get IPs from which pools using a label selector

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "red-pool"
spec:
  blocks:
  - cidr: "20.0.10.0/24"
  serviceSelector:
    matchLabels:
      color: red
```

### Disabling a Pool

IP Pools can be disabled. Disabling a pool will stop LB IPAM from allocating new IPs from the pool, but doesn’t remove existing allocations.

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "blue-pool"
spec:
  blocks:
  - cidr: "20.0.10.0/24"
  disabled: true
```

