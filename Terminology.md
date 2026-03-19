## Identification

For identification purposes, Cilium assigns an internal endpoint id to all endpoints on a cluster node. The endpoint id is unique within the context of an individual cluster node.

## Endpoint Metadata

An endpoint automatically derives metadata from the application containers associated with the endpoint. The metadata can then be used to identify the endpoint for security/policy, load-balancing and routing purposes.


```bash
kubectl -n kube-system exec cilium-zg4jc -- cilium-dbg endpoint list                                       
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])                                                  IPv6   IPv4         STATUS   
           ENFORCEMENT        ENFORCEMENT                                                                                                                   
1435       Disabled           Disabled          26138      k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system          10.0.0.27    ready   
                                                           k8s:io.cilium.k8s.policy.cluster=kubernetes                                                              
                                                           k8s:io.cilium.k8s.policy.serviceaccount=coredns                                                          
                                                           k8s:io.kubernetes.pod.namespace=kube-system                                                              
                                                           k8s:k8s-app=kube-dns                                                                                     
1722       Disabled           Disabled          26138      k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system          10.0.0.129   ready   
                                                           k8s:io.cilium.k8s.policy.cluster=kubernetes                                                              
                                                           k8s:io.cilium.k8s.policy.serviceaccount=coredns                                                          
                                                           k8s:io.kubernetes.pod.namespace=kube-system                                                              
                                                           k8s:k8s-app=kube-dns                                                                                     
2866       Disabled           Disabled          4          reserved:health                                                                     10.0.0.160   ready   
3713       Disabled           Disabled          1          k8s:node-role.kubernetes.io/worker=worker                                                        ready
```


```yaml
apiVersion: cilium.io/v2
kind: CiliumIdentity
metadata:
  creationTimestamp: "2026-02-24T10:14:12Z"
  generation: 1
  labels:
    io.kubernetes.pod.namespace: kube-system
  name: "26138"
  resourceVersion: "5578"
  uid: 06a1c3ff-932e-4213-8322-b3ae37420ef7
  selfLink: /apis/cilium.io/v2/ciliumidentities/26138
security-labels:
  k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name: kube-system
  k8s:io.cilium.k8s.policy.cluster: kubernetes
  k8s:io.cilium.k8s.policy.serviceaccount: coredns
  k8s:io.kubernetes.pod.namespace: kube-system
  k8s:k8s-app: kube-dns
```

```yaml
apiVersion: cilium.io/v2
kind: CiliumEndpoint
metadata:
  creationTimestamp: "2026-02-24T10:14:12Z"
  generation: 1
  labels:
    k8s-app: kube-dns
    pod-template-hash: 7d764666f9
  name: coredns-7d764666f9-8xsc9
  namespace: kube-system
  ownerReferences:
  - apiVersion: v1
    kind: Pod
    name: coredns-7d764666f9-8xsc9
    uid: 86209308-6f00-4d2e-9fff-690924ffef24
  resourceVersion: "5580"
  uid: 76e3fa25-d7cb-41ee-bf59-53767fe8fa56
  selfLink: >-
    /apis/cilium.io/v2/namespaces/kube-system/ciliumendpoints/coredns-7d764666f9-8xsc9
status:
  encryption: {}
  external-identifiers:
    cni-attachment-id: ad485eeb92154b92e0bab1ca6f327b8d8e2fc0e238c801e2cf4e5374fc51c381:eth0
    container-id: ad485eeb92154b92e0bab1ca6f327b8d8e2fc0e238c801e2cf4e5374fc51c381
    k8s-namespace: kube-system
    k8s-pod-name: coredns-7d764666f9-8xsc9
    pod-name: kube-system/coredns-7d764666f9-8xsc9
  id: 1722
  identity:
    id: 26138
    labels:
    - k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=kube-system
    - k8s:io.cilium.k8s.policy.cluster=kubernetes
    - k8s:io.cilium.k8s.policy.serviceaccount=coredns
    - k8s:io.kubernetes.pod.namespace=kube-system
    - k8s:k8s-app=kube-dns
  named-ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: liveness-probe
    port: 8080
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
  - name: readiness-probe
    port: 8181
    protocol: TCP
  networking:
    addressing:
    - ipv4: 10.0.0.129
    node: 91.107.196.196
  service-account: coredns
  state: ready

```


## Identity

All Endpoint are assigned an identity. The identity is what is used to enforce basic connectivity between endpoints. In traditional networking terminology, this would be equivalent to Layer 3 enforcement.

## What is an Identity?

The identity of an endpoint is derived based on the Labels associated with the pod or container which are derived to the endpoint. When a pod or container is started, Cilium will create an endpoint based on the event received by the container runtime to represent the pod or container on the network. As a next step, Cilium will resolve the identity of the endpoint created. Whenever the Labels of the pod or container change, the identity is reconfirmed and automatically modified as required.

After adding a new label to the pod, a new CiliumIdentity will be created and the current CiliumEndpoint will be updated accordingly.

