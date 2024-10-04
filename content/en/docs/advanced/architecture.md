---
title: Technical Architecture
description: Understand LLMariner architecture
---

## Components

LLMariner provisions the LLM stack consisting of the following micro services:

-   Inference Manager
-   Job Manager
-   Model Manager
-   File Manager
-   Vector Store Server
-   User Manager
-   Cluster Manager
-   Session Manager
-   RBAC Manager

Each manager is responsible for the specific feature of LLM services as their names indicate. The following diagram shows the high-level architecture:

{{< img src="images/architecture_diagram.png" >}}

LLMariner has dependency to the following components:

-   Ingress controller
-   SQL database
-   S3-compatible object store
-   [Dex](https://github.com/dexidp/dex)
-   [Milvus](https://milvus.io/)

Ingress controller is required to route traffic to each service. SQL database and S3-compatible object store are used to persist metadata (e.g., fine-tuning jobs), fine-tuned models, and training/validation files. Dex is used to provide authentication.

## Key Technologies

### Autoscaling and Dynamic Model Loading in Inference

Inference Manager dynamically loads models up on requests it receives. It also dynamically auto-scales pods based on demand.

### Session Manager: Secure Access to Kubernetes API Server

LLMariner internally accesses Kubernetes API server to allow end users to access logs of fine-tuning jobs, exec into a Jupyter Notebook, etc. As end users might not have direct access to a Kubernetes API server, LLMariner uses Session Manager to provide a secure tunnel between end users and Kubernetes API server.

Session Manager consists of two components: `server` and `agent`. The `agent` establishes HTTP(S) connections to the `server` and keeps the connections. Upon receiving a request from end users, the `server` forwards the request to the `agent` using one of the established connections. Then the `agent` forwards the request to the Kubernetes API server.

This architecture enables the deployment where the `server` and the `agent` can run in separate Kubernetes clusters. As the `agent` initiates a connection (not the `server`), there is no need to open incoming traffic at the cluster where the `agent` runs. An ingress controller is still the only place where incoming traffic is sent.

### Quota Management for Fine-tuning Jobs

LLMariner allows users to manage GPU quotas with integration with [Kueue](https://github.com/kubernetes-sigs/kueue).
