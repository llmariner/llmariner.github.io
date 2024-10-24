---
title: Set up a Playground (CPU-only)
linkTitle: "Playground (CPU)"
description: Set up the playground environment on the local kind cluster (CPU-only).
weight: 15
---

Following this guide provides you with a simplified, local LLMariner installation by using the Kind and Helm. You can use this simple LLMariner deployment to try out features without GPUs.

{{% alert title="Warn" color="secondary" %}}
Playground environments are for experimentation use only. For a production-ready installation, please refere to the other installation guide.
{{% /alert %}}

## Before you begin

Before you can get started with the LLMariner deployment you must install:

- [kind (Kubernetes in Docker)](https://kind.sigs.k8s.io/docs/user/quick-start)
- [Helmfile](https://helmfile.readthedocs.io/en/latest/#installation)

## Step 1: Clone the repository

To get started, clone the LLMariner repository.

```bash
git clone https://github.com/llmariner/llmariner.git
```

## Step 2: Create a kind cluster

The installation files are in `provision/dev/`. Create a new Kubernetes cluster using kind by running:

```bash
cd provision/dev/
./create_cluster.sh single
```

## Step 3: Install LLMariner

To install LLMariner using helmfile, run the following commands:

```bash
helmfile apply --skip-diff-on-install
```

{{% alert title="Tips" color="primary" %}}
You can filter the components to deploy using the `--selector(-l)` flag. For example, to filter out the monitoring components, set the `-l tier!=monitoring` flag. For deploying just the llmariner, use `-l app=llmariner`.
{{% /alert %}}
