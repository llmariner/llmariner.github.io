---
title: Hosted Control Plane
linkTitle: "Hosting Service"
description: Install just the worker plane and use it with the hosted control plane.
weight: 40
---

CloudNatix provides a hosted control plane of LLMariner.

{{% alert title="Note" color="primary" %}}
Work-in-progress. This is not fully ready yet, and the terms and conditions are subject to change as we might limit the usage based on the number of API calls or the number of GPU nodes.
{{% /alert %}}

CloudNatix provides a hosted control plane of LLMariner. End users can use the full functionality of LLMariner just by registering their worker GPU clusters to this hosted control plane.

## Step 1. Create a CloudNatix account

Create a CloudNatix account if you haven\'t. Please visit <https://app.cloudnatix.com>. You can click one of the \"Sign in or sing up\" buttons for SSO login or you can click \"Sign up\" at the bottom for the email & password login.

## Step 2. Deploy the worker plane components

Deploy the worker plane components LLMariner into your GPU cluster.

The API endpoint of the hosted control plane is <https://api.llm.cloudnatix.com/v1>.

Run `llma auth login` and use the above for the endpoint URL. Then follow `multi_cluster_deployment`{.interpreted-text role="doc"} to obtain a cluster registration key and deploy LLMariner.

TODO: Add an example `values.yaml`.
