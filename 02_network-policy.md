## Overview

Cilium supports multiple policy formats for controlling network traffic:

### 1. Standard Kubernetes NetworkPolicy

Supports Layer 3 (IP) and Layer 4 (Port) policies at ingress or egress of the Pod. This is the native Kubernetes networking policy resource.

### 2. CiliumNetworkPolicy

An extended format available as a CustomResourceDefinition that supports specification of policies at Layers 3-7 (including HTTP) for both ingress and egress. Offers more granular control compared to standard NetworkPolicy.

### 3. CiliumClusterwideNetworkPolicy

A cluster-scoped CustomResourceDefinition for specifying cluster-wide policies to be enforced by Cilium. The specification is the same as CiliumNetworkPolicy with no specified namespace, allowing policies to apply across the entire cluster.