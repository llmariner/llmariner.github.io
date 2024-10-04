---
title: Inference with Open Models
description: >
  Users can run chat completion with open models such as Google Gemma, LLama, Mistral, etc. To run chat completion, users can use the OpenAI Python library, `llma` CLI, or API endpoint.
---

Here is an example chat completion command with the `llma` CLI.

``` bash
llma chat completions create --model google-gemma-2b-it-q4_0 --role user --completion "What is k8s?"
```

If you want to use the Python library, you first need to create an API key:

``` bash
llma auth api-keys create <key name>
```

You can then pass the API key to initialize the OpenAI client and run the completion:

``` python
from openai import OpenAI

client = OpenAI(
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

You can also just call ``client = OpenAI()` if you set environment variables `OPENAI_BASE_URL` and `OPENAI_API_KEY`.

If you want to hit the API endpoint directly, you can use `curl`. Here is an example.

``` bash
curl \
  --request POST \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{"model": "google-gemma-2b-it-q4_0", "messages": [{"role": "user", "content": "What is k8s?"}]}' \
  http://localhost:8080/v1/chat/completions
```

Please see [the fine-tuning page](./fine_tuning.html) if you want to generate a fine-tuning model and use that for chat completion.
