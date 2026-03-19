## modes

1. Encapsulation
2. Native Routing

### Encapsulation (tunnel) (default)
if our nodes have physical connectivity to each other.

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

```yaml
routingMode: "native"
ipv4NativeRoutingCIDR: "10.244.0.0/16"
```


##### Pros
- Better performance
- More efficient use of MTU
- Easier debugging
- Optimized for large-scale deployments

##### Cons
- requires for underlay network changes
- complex routing configuration
- no native Cloud dependecy


