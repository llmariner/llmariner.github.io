---
title: High-Level Architecture
linkTitle: How works?
description: An overview of the key components that make up a LLMarinr.
weight: 20
---

This page provides a high-level overview of the essential components that make up a LLMariner:

<!-- original file is located at diagram/llmariner.excalidraw -->
{{< img src="/images/highlevel_architecture.png" >}}

## Overall Design

LLMariner consists of a control-plane and one or more worker-planes:

Control-Plane components
: Expose the OpenAI-compatible APIs and manage the overall state of LLMariner and receive a request from the client.

Worker-Plane components
: Run every worker clusters, process tasks using compute resources such as GPUs in response to requests from the control-plane.

## Core Components

Here's a brief overview of the main components:

Inference Manager
: Manage inference runtimes (e.g., vLLM and Ollama) in containers, load models, and process requests. Also, auto-scale runtimes based on the number of in-flight requests.

Job Manager
: Run fine-tuning or training jobs based on requests, and launch Jupyter Notebooks.

Session Manager
: Forwards requests from the client to the worker cluster that need the Kubernetes API, like displaying Job logs.

Data Managers
: Manage models, files, and vector data for RAG.

Auth Managers
: Manage information such as users, organizations, and clusters, and perform authentication and role-based access control for API requests.

## What's Next

- Ready to [Get Started]({{<ref "setup">}})?
- Take a look at the [Technical Details]({{<ref "architecture">}})
- Take a look at LLMariner core [Features]({{<ref "features">}})
