---
title: GPU Federation
linkTitle: GPU Federation
description: Users can create a global pool of GPUs across multiple clusters and efficiently utilize them.
weight: 8
---

## Overview

LLMariner GPU federation creates a global pool of GPUs across multiple
clusters so that end users can submit training jobs without knowing
GPU availability of individual K8s clusters.

End users can continue to use their existing tools to interact with
the global GPU pool. For example, a user can create a new job with
`kubectl create`. LLMariner schedules it one of the worker clusters
where a sufficient number of GPUs are available, and pods will start
running there.

Please see [this demo video](https://vimeo.com/1057981914) for an example job submissions flow.

## Install Procedure

### Prerequisites

The following K8s clusters are required:

- A k8s cluster for running the LLMariner control plane
- Worker k8s clusters with GPU nodes
- A k8s cluster that works as an aggregation point for the worker clusters (Hereafter we call this *proxy cluster*)

The proxy cluster does not require GPU nodes.

### Step 1. Install the LLMariner control plane and worker plane

Follow [multi_cluster_deployment](../setup/install/multi_cluster_production/) and install the LLMariner control plane and worker plane.

### Step 2. Create a service account API key and a k8s secret

Run the following command to create an API key for the service account.

```bash
llma auth api-keys create key-for-proxy-cluster --service-account --role tenant-system
```

We use this API key in the proxy cluster. Run the following command to create a K8s secret
in the proxy cluster.

```bash
kubectl create namesapce llmariner
kubectl create secret -n llmariner generic syncer-api-key --from-literal=key=<API key secret>
```

### Step 3. Install `job-manager-syncer` in the proxy cluster.

Run:

```bash
helm upgrade \
  --install \
  -n llmariner \
  llmariner \
  oci://public.ecr.aws/cloudnatix/llmariner-charts/llmariner \
  -f ./values.yaml
```

Here is the example `values.yaml`.

```yaml
tags:
  # Set these to only deploy job-manager-syncer
  control-plane: false
  worker: false
  tenant-control-plane: true

job-manager-syncer:
  # The gRPC endpoint of LLMariner control plane (same value as `global.worker.controlPlaneAddr` in the worker plane config).
  jobManagerServerSyncerServiceAddr: control-plane:80
  # The base URL of the LLMariner API endpoint.
  sessionManagerEndpoint: http://control-plane/v1
  tenant:
    # k8s secret name and key that contains the LLMariner API key secret.
    apiKeySecret:
      name: syncer-api-key
      key: key
```
