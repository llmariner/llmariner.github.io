---
title: dify
description: Integrate with Dify for LLM application development.
weight: 6
---

[Dify](https://dify.ai/) is is an open-source LLM app development platform.
It can orchestrate LLM apps from agents to complex AI workflows, with an RAG engine.

You can add LLMariner as one of Dify's model providers with the following steps:

1. Click the user profile icon.
2. Click "Settings"
3. Click "Model Provider"
4. Search "OpenAI-API-compatible" and click "Add model"
5. Configure a model name, API key,a nd API endpoint URL.

{{< img src="/images/dify.png" >}}

You can then use the registered model from your LLM applications. For example, you
can create a new application by "Create from Template" and replace the use of an OpenAI
model with the configured model.

If you want to deploy Dify in your Kubernetes clusters, follow
[README.md](https://github.com/langgenius/dify/tree/716576043de3bdaab5aeac28a83d31bb054f8ec4?tab=readme-ov-file#advanced-setup)
in the Dify GitHub repository.
