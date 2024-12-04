---
title: Aider
description: Integrate with Aider for AI pair programming
weight: 3
---

[Aider](https://aider.chat/) is AI pair programming in your terminal or browser.

Aider supports the OpenAI compatible API, and you can configure the endpoint
and the API key with environment variables.


Here is an example installation and configuration procedure.

```bash
python -m pip install -U aider-chat

export OPENAI_API_BASE=<Base URL (e.g., http://localhost:8080/v1)>
export OPENAI_API_KEY=<API key>
```

You can then run Aider in your terminal or browser. Here is an example command
that launches Aider in your browser with Llama 3.1 70B.

```bash
<Move to your github repo directory>

aider --model openai/meta-llama-Meta-Llama-3.1-70B-Instruct-awq --browser
```

Please note that the model name requires the `openai/` prefix.

https://aider.chat/examples/README.html has example chat transcripts
for building applications (e.g., "make a flask app with a /hello endpoint that returns hello world").
