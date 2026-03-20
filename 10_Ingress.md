## Kubernetes Ingress

Cilium uses the standard Kubernetes Ingress resource definition, with an `ingressClassName` of `cilium`.

* The ingress controller creates a Service of `LoadBalancer` type, so your environment will need to support this.

## Load Balancer Modes for the Ingress Resource

There are two load balancer modes available:

- **Dedicated:** The Ingress controller will create a dedicated loadbalancer for the Ingress.
- **Shared:** The Ingress controller will use a shared loadbalancer for all Ingress resources.


### What Dedicated vs Shared Modes Mean in Cilium Ingress

#### Dedicated Mode
* each Ingress gets its own LoadBalancer Service and its own external IP/FQDN.
* Cilium will create a separate Service of type LoadBalancer for each Ingress resource. That means a unique LB IP per Ingress.

- Example:
If you have 3 Ingresses, you get 3 distinct LoadBalancer Services and 3 external IPs.

When it’s useful
* You want strict isolation.
* You want separate IPs because you don’t want conflicts in host/path routing configuration.
* You want different annotations, LB settings, specific IPs per Ingress.

Drawbacks
* Consumes more resources (external IPs, LoadBalancer services).
* Might increase cloud load balancer costs if running on cloud.


✅ Shared mode
*  All Ingress resources share `one single LoadBalancer Service.`
The Cilium Ingress controller will configure that one single LB Service and route traffic for all Ingress objects through it.

How it works conceptually
* Only one Kubernetes Service of type LoadBalancer exists (e.g., cilium-ingress).
* That LB gets one external IP.
* All the Ingress rules you create are combined logically under that one IP/LB.

Benefits
* Saves external IPs and cloud LB resources.
* Useful if you have limited IPs or want a single entry point.

Drawbacks
* You must carefully manage host/path conflicts yourself.
* All traffic comes to one IP, so you rely fully on Ingress routing logic.


## Prerequisites

1. Cilium must be configured with the kube-proxy replacement, using kubeProxyReplacement=true. For more information, see kube-proxy replacement.
2. Cilium must be configured with the L7 proxy enabled using `l7Proxy=true (enabled by default).`
3. By default, the Ingress controller creates a Service of LoadBalancer type, so your environment will need to support this. Alternatively, you can change this to NodePort or, since Cilium 1.16+, directly expose the Cilium L7 proxy on the host network.

```bash
helm upgrade cilium oci://quay.io/cilium/charts/cilium --version 1.19.1 \
   --namespace kube-system \
   --reuse-values \
   --set ingressController.enabled=true \
   --set ingressController.default=true \
   --set ingressController.loadbalancerMode=dedicated
```
```bash
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
```


```bash
cilium status
```

## How Cilium Ingress and Gateway API differ from other Ingress controllers
One of the biggest differences between Cilium’s Ingress and Gateway API support and other Ingress controllers is how closely tied the implementation is to the CNI. **For Cilium, Ingress and Gateway API are part of the networking stack**, and so behave in a different way to other Ingress or Gateway API controllers (even other Ingress or Gateway API controllers running in a Cilium cluster).


## Cilium’s ingress config and CiliumNetworkPolicy

for ingress config, there’s also an additional step. Traffic that arrives at Envoy for Ingress or Gateway API is assigned the special ingress identity in Cilium’s Policy engine.

Traffic coming from outside the cluster is usually assigned the `world` identity (unless there are IP CIDR policies in the cluster). This means that there are actually `two` logical Policy enforcement points in Cilium Ingress - **before traffic arrives at the ingress identity, and after, when it is about to exit the per-node Envoy.**

![alt text](https://docs.cilium.io/en/stable/_images/ingress-policy.png)

This means that, when applying Network Policy to a cluster, it’s important to ensure that both steps are allowed, and that traffic is allowed `from world to ingress`, and `from ingress to identities in the cluster`.


## Host Network Mode

Host network mode allows you to expose the Cilium ingress controller (Envoy listener) directly on the host network. This is useful in cases **where a LoadBalancer Service is unavailable, such as in development environments or environments with cluster-external loadbalancers.**

**Note:** Enabling the Cilium ingress controller host network mode automatically disables the LoadBalancer/NodePort type Service mode.

### To Enable Host Network Mode

```yaml
ingressController:
  enabled: true
  hostNetwork:
    enabled: true
```


## Example Configuration on Bare Metal / Cloud Infrastructure

```bash
cilium upgrade --set ingressController.enabled=true \
 --set ingressController.default=true \
 --set ingressController.loadbalancerMode=shared  \
 --reuse-values
```

For cloud environments with LoadBalancer support:
- Install the appropriate Cloud Controller Manager (e.g., Hetzner CCM for Hetzner Cloud)
- Configure load balancer targets with public IPs from the cloud provider's console


```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  namespace: default
spec:
  ingressClassName: cilium
  rules:
  - host: productpage.example.com
    http:
      paths:
      - backend:
          service:
            name: details
            port:
              number: 9080
        path: /details
        pathType: Prefix
      - backend:
          service:
            name: productpage
            port:
              number: 9080
        path: /
        pathType: Prefix
```


```bash
curl --fail -s http://productpage.example.com/details/1 | jq
{
  "id": 1,
  "author": "William Shakespeare",
  "year": 1595,
  "type": "paperback",
  "pages": 200,
  "publisher": "PublisherA",
  "language": "English",
  "ISBN-10": "1234567890",
  "ISBN-13": "123-1234567890"
}
```


## Testing Block External Network Policy

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "external-lockdown"
spec:
  description: "Block all the traffic originating from outside of the cluster"
  endpointSelector: {}
  ingress:
  - fromEntities:
    - cluster
```


```bash
curl --fail -v http://productpage.achaemenid.nl/details/1 | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Host productpage.achaemenid.nl:80 was resolved.
* IPv6: 2606:4700:3034::ac43:de93, 2606:4700:3035::6815:2674
* IPv4: 188.114.97.0, 188.114.96.0
*   Trying [2606:4700:3034::ac43:de93]:80...
* Connected to productpage.achaemenid.nl (2606:4700:3034::ac43:de93) port 80
> GET /details/1 HTTP/1.1
> Host: productpage.example.com
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 403 Forbidden
< Date: Fri, 27 Feb 2026 17:38:59 GMT
< Content-Type: text/plain
< Content-Length: 15
< Connection: keep-alive
< server: cloudflare
< cf-cache-status: DYNAMIC
< Nel: {"report_to":"cf-nel","success_fraction":0.0,"max_age":604800}
< Report-To: {"group":"cf-nel","max_age":604800,"endpoints":[{"url":"https://a.nel.cloudflare.com/report/v4?s=fdU6nz3KhWFCJi7lZA206Ar2rS%2FcBpGKXXrvqKzNrX75pHsXeDj6%2Fgm5WDJtEHtvMnM%2FLc28S73Gm%2BwefNxXk9Nj2s%2BmYdOyKEoMv6uMMkUWDJAiFGRVIfE%2B4ociVs%2BXqrW78NA%3D"}]}
< CF-RAY: 9d49757f9b12ef02-AMS
< alt-svc: h3=":443"; ma=86400
< 
* The requested URL returned error: 403
  0    15    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
* Closing connection
curl: (22) The requested URL returned error: 403
```

---
### Network Policies

### Default Deny Ingress Policy

```yaml
---
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "default-deny"
spec:
  description: "Block all the traffic (except DNS) by default"
  egress:
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: kube-system
        k8s-app: kube-dns
    toPorts:
    - ports:
      - port: '53'
        protocol: UDP
      rules:
        dns:
        - matchPattern: '*'
  endpointSelector:
    matchExpressions:
    - key: io.kubernetes.pod.namespace
      operator: NotIn
      values:
      - kube-system

```

## TLS Configuration

### Install Cert-Manager

```bash
helm -n cert-manager install cert-manager --set crds.enabled=true --values values.yaml --values override.yaml . --create-namespace
```

### Create DNS Provider Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
data:
  api-token: <Edit-Zone-mydomain-TOKEN-API>
type: Opaque
```

### Create ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
        # For other DNS providers, adjust accordingly (e.g., Route53, Google Cloud DNS, etc.)
```

### Create TLS Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: cilium
  rules:
  - host: hipstershop.example.com
    http:
      paths:
      - backend:
          service:
            name: productcatalogservice
            port:
              number: 3550
        path: /hipstershop.ProductCatalogService
        pathType: Prefix
      - backend:
          service:
            name: currencyservice
            port:
              number: 7000
        path: /hipstershop.CurrencyService
        pathType: Prefix
  - host: bookinfo.example.com
    http:
      paths:
      - backend:
          service:
            name: details
            port:
              number: 9080
        path: /details
        pathType: Prefix
      - backend:
          service:
            name: productpage
            port:
              number: 9080
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - bookinfo.example.com
    - hipstershop.example.com
    secretName: demo-cert
```


```bash
curl https://bookinfo.example.com/details/1
{"id":1,"author":"William Shakespeare","year":1595,"type":"paperback","pages":200,"publisher":"PublisherA","language":"English","ISBN-10":"1234567890","ISBN-13":"123-1234567890"}
```

---

## Defaults certificate for Ingresses

Cilium can use a default certificate for ingresses without .spec.tls[].secretName set. It’s still necessary to have .spec.tls[].hosts defined.

```bash
helm upgrade cilium cilium/cilium --version 1.19.1 \
   --namespace kube-system \
   --reuse-values \
   --set ingressController.defaultSecretNamespace=kube-system \
   --set ingressController.defaultSecretName=default-cert
```