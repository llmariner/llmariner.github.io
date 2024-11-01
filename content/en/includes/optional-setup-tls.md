---
draft: true
---

First follow the [cert-manager installation document](https://cert-manager.io/docs/installation/) and install cert-manager to your K8s cluster if you don't have one. Then create a `ClusterIssuer` for your domain. Here is an example manifest that uses Let\'s Encrypt.

``` yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: user@mydomain.com
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
       ingress:
          ingressClassName: kong
    - selector:
        dnsZones:
        - llm.mydomain.com
      dns01:
        ...
```

Then you can add the following to `values.yaml` of LLMariner to enable TLS.

``` yaml
global:
  ingress:
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt
    tls:
      hosts:
      - api.llm.mydomain.com
      secretName: api-tls
```

The ingresses created from the Helm chart will have the following annotation and spec:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
...
spec:
  tls:
  - hosts:
    - api.llm.mydomain.com
    secretName: api-tls
  ...
```
