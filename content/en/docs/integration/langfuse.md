---
title: Langfuse
description: Integrate with Langfuse for LLM engineering.
weight: 7
---

[Langfuse](https://github.com/langfuse/langfuse) is an open source LLM engineering platform. You can integrate Langfuse
with LLMariner as Langfuse provides an SDK for the OpenAI API.

Here is an example procedure for running Langfuse locally:

```bash
git clone https://github.com/langfuse/langfuse.git
cd langfuse
docker compose up -d
```

You can sign up and create your account. Then you can generate API keys
and put them in environmental variables.

```bash
export LANGFUSE_SECRET_KEY=...
export LANGFUSE_PUBLIC_KEY=...
export LANGFUSE_HOST="http://localhost:3000"
```

You can then use `langfuse.openai` instead of `openai` in your Python scripts
to record traces in Langfuse.

```python
from langfuse.openai import openai

client = openai.OpenAI(
  base_url="<Base URL (e.g., http://localhost:8080/v1)>",
  api_key="<API key secret>"
)

completion = client.chat.completions.create(
  model="google-gemma-2b-it-q4_0",
  messages=[
    {"role": "user", "content": "What is k8s?"}
  ],
  stream=True
)
for response in completion:
  print(response.choices[0].delta.content, end="")
print("\n")
```

Here is an example screenshot.

{{< img src="/images/langfuse.png" >}}
