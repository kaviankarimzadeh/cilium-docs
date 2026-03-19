## install demo apps
[Cilium demo app](./demo-app.md)

### Apply an L3/L4 Policy

![https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_gsg.png](https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_gsg.png)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

```bash
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/minikube/sw_l3_l4_policy.yaml
```

The above policy whitelists traffic sent from any pods with label (org=empire) to deathstar pods with label (org=empire, class=deathstar) on TCP port 80.


```bash
#only the tiefighter pods with the label org=empire will succeed
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

#same request run from an xwing pod will fail - press Control-C to kill the curl request
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing     
^C
```


### Inspecting the Policy

If we run cilium-dbg endpoint list again we will see that the pods with the label org=empire and class=deathstar now have ingress policy enforcement enabled as per the policy above

```bash
kubectl -n kube-system exec cilium-tqzvw  -- cilium-dbg endpoint list
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])                                              IPv6   IPv4          STATUS   
           ENFORCEMENT        ENFORCEMENT                                                                                                                
84         Disabled           Disabled          41726      k8s:app.kubernetes.io/name=tiefighter                                           10.244.1.42   ready   
                                                           k8s:class=tiefighter                                                                                  
                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default                                
                                                           k8s:io.cilium.k8s.policy.cluster=kind-cilium-cluster                                                  
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                       
                                                           k8s:io.kubernetes.pod.namespace=default                                                               
                                                           k8s:org=empire                                                                                        
1663       Disabled           Disabled          1          reserved:host                                                                                 ready   
1974       Enabled            Disabled          19452      k8s:app.kubernetes.io/name=deathstar                                            10.244.1.83   ready   
                                                           k8s:class=deathstar                                                                                   
                                                           k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default                                
                                                           k8s:io.cilium.k8s.policy.cluster=kind-cilium-cluster                                                  
                                                           k8s:io.cilium.k8s.policy.serviceaccount=default                                                       
                                                           k8s:io.kubernetes.pod.namespace=default                                                               
                                                           k8s:org=empire                                                                                        
3702       Disabled           Disabled          4          reserved:health                                                                 10.244.1.20   ready
```

### Apply and Test HTTP-aware L7 Policy
For example, consider that the deathstar service exposes some maintenance APIs which should not be called by random empire ships. To see this run:

```bash
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Panic: deathstar exploded

goroutine 1 [running]:
main.HandleGarbage(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /code/src/github.com/empire/deathstar/
        temp/main.go:9 +0x64
main.main()
        /code/src/github.com/empire/deathstar/
        temp/main.go:5 +0x85
```

![alt text](https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_l7_gsg.png)

Here is an example policy file that extends our original policy by limiting tiefighter to making only a POST /v1/request-landing API call, but disallowing all other calls (including PUT /v1/exhaust-port).

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/minikube/sw_l3_l4_l7_policy.yaml
```

```bash
➜  kind kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed
➜  kind kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied
```

### observe the L7 policy

```bash
kubectl -n kube-system exec cilium-tqzvw -- cilium-dbg policy get
```

###  monitor the HTTP requests live

```bash
kubectl exec -it -n kube-system cilium-tqzvw -- cilium-dbg monitor -v --type l7
```