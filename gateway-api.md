## Overview

Cilium Gateway API provides a modern, standards-based approach to ingress and load balancing using the Kubernetes Gateway API. It offers superior flexibility and control compared to traditional Ingress resources, including HTTP routing, TLS termination, and traffic management.

## Prerequisites

Cilium must be configured with the kube-proxy replacement, using kubeProxyReplacement=true. For more information, see kube-proxy replacement.
Cilium must be configured with the L7 proxy enabled using l7Proxy=true (enabled by default).
The below CRDs from Gateway API v1.4.1 must be pre-installed. Please refer to the documentation for installation steps. Alternatively, the below snippet could be used.

## Installation

```bash
$ cilium install --version 1.19.1 --set kubeProxyReplacement=true --set gatewayAPI.enabled=true
```


## Host network mode

Host network mode allows you to expose the Cilium Gateway API Gateway directly on the host network. This is useful in cases where a LoadBalancer Service is unavailable, such as in development environments or environments with cluster-external loadbalancers.

```yaml
gatewayAPI:
  enabled: true
  hostNetwork:
    enabled: true
```


## Example Configuration on Bare Metal / Cloud Infrastructure

### Disable Cilium Ingress Controller 

```bash
cilium upgrade --set ingressController.enabled=false --set ingressController.default=false  --reuse-values
```

### Enable Gateway API

```bash
cilium upgrade --set gatewayAPI.enabled=true  --reuse-values
```

Verify GatewayClass:

```bash
kubectl get gatewayclass
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       5m49s
```


### Certificate Setup

Configure Cert-Manager ClusterIssuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-wildcard
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-wildcard-account-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
      selector:
        dnsZones:
        - "example.com"
```

Create wildcard certificate:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-certificate
  namespace: kube-system
spec:
  secretName: wildcard-certificate-tls
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-wildcard
  dnsNames:
    - "*.example.com"
```

### Create Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gw
  namespace: kube-system
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.example.com"
    allowedRoutes:
      namespaces:
        from: All

  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.example.com"
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      certificateRefs:
      - name: wildcard-certificate-tls
```

Verify Gateway creation:

```bash
kubectl get gatewayclass
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       27m

kubectl get gateway -A
NAMESPACE     NAME        CLASS    ADDRESS         PROGRAMMED   AGE
kube-system   public-gw   cilium   128.140.29.20   True         3m44s

kubectl get svc -n kube-system | grep LoadBalancer
cilium-gateway-public-gw   LoadBalancer   10.100.60.166   10.0.1.5,128.140.29.20,2a01:4f8:c011:3b1::1   80:30583/TCP,443:31976/TCP   4m20s
```

### Create HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: productpage
  namespace: default
spec:
  parentRefs:
    - name: public-gw
      namespace: kube-system
  hostnames:
    - productpage.example.com
  rules:
  - backendRefs:
    - name: productpage
      port: 9080
    matches:
    - path:
        type: PathPrefix
        value: /
  - backendRefs:
    - name: details
      port: 9080
    matches:
    - path:
        type: PathPrefix
        value: /details
```

### Test the HTTPRoute

```bash
curl https://productpage.example.com/details/1                  
{"id":1,"author":"William Shakespeare","year":1595,"type":"paperback","pages":200,"publisher":"PublisherA","language":"English","ISBN-10":"1234567890","ISBN-13":"123-1234567890"}
```

Verify HTTPRoute status:

```bash
kubectl get httproute -A
NAMESPACE   NAME          HOSTNAMES                       AGE
default     productpage   ["productpage.example.com"]   10m

kubectl describe httproute productpage 
Name:         productpage
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         HTTPRoute
Metadata:
  Creation Timestamp:  2026-03-02T12:09:05Z
  Generation:          2
  Resource Version:    839362
  UID:                 d56f0916-e662-447d-ac01-5ad329d398aa
Spec:
  Hostnames:
    productpage.achaemenid.nl
  Parent Refs:
    Group:      gateway.networking.k8s.io
    Kind:       Gateway
    Name:       public-gw
    Namespace:  kube-system
  Rules:
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    productpage
      Port:    9080
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /
    Backend Refs:
      Group:   
      Kind:    Service
      Name:    details
      Port:    9080
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /details
Status:
  Parents:
    Conditions:
      Last Transition Time:  2026-03-02T12:18:05Z
      Message:               Accepted HTTPRoute           ### Accepted: The HTTPRoute configuration was correct and accepted.
      Observed Generation:   2
      Reason:                Accepted                    
      Status:                True
      Type:                  Accepted
      Last Transition Time:  2026-03-02T12:18:05Z
      Message:               Service reference is valid
      Observed Generation:   2
      Reason:                ResolvedRefs                 ### ResolvedRefs: The referenced services were found and are valid references.
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         io.cilium/gateway-controller
    Parent Ref:
      Group:      gateway.networking.k8s.io
      Kind:       Gateway
      Name:       public-gw
      Namespace:  kube-system
Events:           <none>
```