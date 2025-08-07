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

### Step 2.1 Generate a cluster registration key

A cluster registration key is required to register your cluster.

If you prefer GUI, visit https://app.llm.cloudnatix.com/app/admin and click "Add worker cluster".

If you prefer CLI, run `llma auth login` and use the above for the endpoint URL. Then run:

``` bash
llma admin clusters register <cluster-name>
```

Then create a secret containing the registration key.

``` bash
REGISTRATION_KEY=clusterkey-...

kubectl create secret generic \
  -n llmariner \
  cluster-registration-key \
  --from-literal=regKey="${REGISTRATION_KEY}"
```

### Step 2.2 Create a k8s secret for Hugging Face API key (optional)

If you want to download models from Hugging Face, create a k8s secret with the following command:

```bash
HUGGING_FACE_HUB_TOKEN=...

kubectl create secret generic \
  huggingface-key \
  -n llmariner \
  --from-literal=apiKey=${HUGGING_FACE_HUB_TOKEN}
```

### Step 2.3 Set up an S3-compatible object store

You can use AWS S3, MinIO, or other S3-compatible object store.

Please see [create an S3 bucket](./single_cluster_eks/#step-3-create-an-s3-bucket) for the example AWS S3 bucket configuration.


### Step 2.4 Deploy LLMariner

Run

``` bash
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install \
  --namespace llmariner \
  --create-namespace \
  llmariner oci://public.ecr.aws/cloudnatix/llmariner-charts/llmariner \
  --values <values.yaml>
```


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

  # Update this to your S3 object store.
  objectStore:
    s3:
      bucket: cloudnatix-installation-demo
      endpointUrl: ""
      region: us-west-2

  # Update this to the k8s secret required to access the S3 object store.
  awsSecret:
    # Name of the secret.
    name: aws
    # Secret key for the access key ID.
    accessKeyIdKey: accessKeyId
    # Secret key for the secret access key.
    secretAccessKeyKey: secretAccessKey

inference-manager-engine:
  inferenceManagerServerWorkerServiceAddr: inference.llm.cloudnatix.com:443
  replicaCount: 2
  runtime:
    runtimeImages:
      ollama: mirror.gcr.io/ollama/ollama:0.6.3-rc0
      vllm: mirror.gcr.io/vllm/vllm-openai:v0.8.5
      triton: nvcr.io/nvidia/tritonserver:24.09-trtllm-python-py3
  engineHeartbeat:
    reconnectOnNoHeartbeat: true
    heartbeatTimeout: 120s
  model:
    default:
      runtimeName: vllm
    overrides:
      meta-llama/Llama-3.2-1B-Instruct:
        preloaded: false
        runtimeName: vllm
        resources:
          limits:
            nvidia.com/gpu: 1

model-manager-loader:
  baseModels:
  - meta-llama/Llama-3.2-1B-Instruct
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
