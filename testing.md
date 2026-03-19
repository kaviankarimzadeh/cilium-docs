
## test cilium network policy

![alt text](https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_gsg.png)

```
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/minikube/http-sw-app.yaml
kubectl get pods,svc
```

```
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

```
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/minikube/sw_l3_l4_policy.yaml
```


```
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

```
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
```
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

If we run cilium-dbg endpoint list again we will see that the pods with the label org=empire and class=deathstar now have ingress policy enforcement enabled as per the policy above.


kubectl get cnp
kubectl describe cnp rule1


## Apply and Test HTTP-aware L7 Policy

kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port

![alt text](https://docs.cilium.io/en/stable/_images/cilium_http_l3_l4_l7_gsg.png)

Cilium is capable of enforcing HTTP-layer (i.e., L7) policies to limit what URLs the tiefighter is allowed to reach. Here is an example policy file that extends our original policy by limiting tiefighter to making only a POST /v1/request-landing API call, but disallowing all other calls (including PUT /v1/exhaust-port).

```
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

```
➜  ~ kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

➜  ~ kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied
```

Note that path matches the exact url, if for example you want to allow anything under /v1/, you need to use a regular expression

```
path: "/v1/.*"
```


It is also possible to monitor the HTTP requests live by using cilium-dbg monitor


## clean up
```
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.19.1/examples/minikube/http-sw-app.yaml


kubectl delete cnp rule1
```