---
title: Hosted Control Plane
linkTitle: "Hosted Control Plane"
description: Install just the worker plane and use it with the hosted control plane.
weight: 40
---

CloudNatix provides a hosted control plane of LLMariner.

{{% alert title="Note" color="primary" %}}
Work-in-progress. This is not fully ready yet, and the terms and conditions are subject to change as we might limit the usage based on the number of API calls or the number of GPU nodes.
{{% /alert %}}

CloudNatix provides a hosted control plane of LLMariner. End users can use the full functionality of LLMariner just by registering their worker GPU clusters to this hosted control plane.

## Step 1. Create a CloudNatix account

Create a CloudNatix account if you haven\'t. Please visit <https://app.cloudnatix.com>. You can click one of the \"Sign in or sign up\" buttons for SSO login or you can click \"Sign up\" at the bottom for the email & password login.

## Step 2. Deploy the worker plane components

Deploy the worker plane components of LLMariner into your GPU cluster.

The API endpoint of the hosted control plane is <https://api.llm.cloudnatix.com/v1>.

Run `llma auth login` and use the above for the endpoint URL. Then follow [multi_cluster_deployment](./multi_cluster_production/#deploying-control-plane-components) to obtain a cluster registration key, create a k8s secret for the registration key, and deploy LLMariner.

Here is an example `values.yaml`:

```yaml
tags:
  control-plane: false

global:
  worker:
    controlPlaneAddr: api.llm.cloudnatix.com:443
    tls:
      enable: true
    registrationKeySecret:
      name: cluster-registration-key
      key: regKey

  objectStore:
    s3:
      bucket: cloudnatix-installation-demo
      endpointUrl: ""
      region: us-west-2

  awsSecret:
    name: aws
    accessKeyIdKey: accessKeyId
    secretAccessKeyKey: secretAccessKey

inference-manager-engine:
  inferenceManagerServerWorkerServiceAddr: inference.llm.cloudnatix.com:443
  replicaCount: 2
  runtime:
    runtimeImages:
      ollama: mirror.gcr.io/ollama/ollama:0.3.6
      vllm: public.ecr.aws/cloudnatix/llm-operator/vllm-openai:v0.8.5.post1
      triton: nvcr.io/nvidia/tritonserver:24.09-trtllm-python-py3
  model:
    default:
      runtimeName: vllm
    overrides:
      NikolayKozloff/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M-GGUF:
        preloaded: true
        runtimeName: vllm
        resources:
          limits:
            nvidia.com/gpu: 1
        vllmExtraFlags:
        - --tokenizer
        - deepseek-ai/DeepSeek-R1-Distill-Qwen-14B
      lmstudio-community/phi-4-GGUF/phi-4-Q4_K_M.gguf:
        preloaded: true
        runtimeName: vllm
        resources:
          limits:
            nvidia.com/gpu: 1
        vllmExtraFlags:
        - --tokenizer
        - microsoft/phi-4

model-manager-loader:
  baseModels:
  - lmstudio-community/phi-4-GGUF/phi-4-Q4_K_M.gguf
  - NikolayKozloff/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M-GGUF
  downloader:
    kind: huggingFace
    huggingFace:
      cacheDir: /tmp/.cache/huggingface/hub
  huggingFaceSecret:
    name: huggingface-key
    apiKeyKey: apiKey

job-manager-dispatcher:
  notebook:
    llmarinerBaseUrl: https://api.llm.cloudnatix.com/v1

session-manager-agent:
  sessionManagerServerWorkerServiceAddr: session.llm.cloudnatix.com:443
```
