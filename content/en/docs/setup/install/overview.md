---
title: Install with Helm
linkTitle: "Overview"
description: Install LLMariner with Helm.
weight: 10
---

## Prerequisites

LLMariner requires the following resources:

-   [Nvidia GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
-   Ingress controller (to route API requests)
-   SQL database (to store jobs/models/files metadata)
-   S3-compatible object store (to store training files and models)
-   [Milvus](https://milvus.io/) (for RAG, optional)

LLMariner can process inference requests on CPU nodes, but it can be best used with GPU nodes. Nvidia GPU Operator is required to install the device plugin and make GPUs visible in the K8s cluster.

Preferably the ingress controller should have a DNS name or an IP that is reachable from the outside of the EKS cluster. If not, you can rely on port-forwarding to reach the API endpoints.

{{% alert title="Note" color="primary" %}}
When port-forwarding is used, the same port needs to be used consistently as the port number will be included the OIDC issuer URL. We will explain details later.
{{% /alert %}}

You can provision RDS and S3 in AWS, or you can deploy Postgres and [MinIO](https://min.io/) inside your EKS cluster.

## Install with Helm

We provide a Helm chart for installing LLMariner. You can obtain the Helm chart from our repository and install.

``` bash
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install \
  --namespace <namespace> \
  --create-namespace \
  llmariner oci://public.ecr.aws/cloudnatix/llmariner-charts/llmariner \
  --values <values.yaml>
```

Once installation completes, you can interact with the API endpoint using the [OpenAI Python library](https://github.com/openai/openai-python), running our CLI, or directly hitting the endpoint. To download the CLI, run:

{{< include "../../../includes/cli-install.md" >}}
