---
title: k8sgpt
description: Integrate with k8sgpt to diagnose and triage issues in your Kubernetes clusters.
weight: 5
---

[k8sgpt](https://github.com/k8sgpt-ai/k8sgpt) is a tool for scanning your
Kubernetes clusters, diagnosing, and triaging issues in simple English.

You can use LLMariner as a backend of k8sgpt by running the following command:

```bash
k8sgpt auth add \
  --backend openai \
  --baseurl <LLMariner base URL (e.g., http://localhost:8080/v1/) \
  --password <LLMariner API Key> \
  --model <Model ID>
```

Then you can a command like `k8sgpt analyze` to inspect your Kubernetes cluster.

```bash
k8sgpt analyze --explain
```
