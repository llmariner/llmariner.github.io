---
draft: true
---

An ingress controller is required to route HTTP/HTTPS requests to the LLMariner components. Any ingress controller works, and you can skip this step if your EKS cluster already has an ingress controller.

Here is an example that installs [Kong](https://konghq.com/) and make the ingress controller reachable via AWS loadbalancer:

``` bash
helm repo add kong https://charts.konghq.com
helm repo update
helm upgrade --install --wait \
  --namespace kong \
  --create-namespace \
  kong-proxy kong/kong \
  --set proxy.annotations.service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout=300 \
  --set ingressController.installCRDs=false \
  --set fullnameOverride=false
```
