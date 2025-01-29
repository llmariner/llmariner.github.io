---
title: n8n
description: Integrate with n8n and get the web UI for the AI assistant.
weight: 7
---

[n8n](https://n8n.io/) is a no-code platform for workload automation. You can deploy
n8n to your Kubernetes clusters. Here is an example command:

``` bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n
  namespace: n8n
spec:
  selector:
    matchLabels:
      name: n8n
  template:
    metadata:
      labels:
        name: n8n
    spec:
      containers:
      - name: n8n
        image: docker.n8n.io/n8nio/n8n
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: n8n
  namespace: n8n
spec:
  type: ClusterIP
  selector:
    name: n8n
  ports:
  - port: 5678
    name: http
    targetPort: http
    protocol: TCP
EOF
```

You can then access n8n with port forwarding:

``` bash
kubectl port-forward -n n8n service/n8n 5678
```

You can create an "OpenAI Model" node and configure its base URL and credential to hit LLMariner.
