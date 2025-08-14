---
title: Inference with Open Models
linkTitle: Inference
description: >
  Users can run chat completion with open models such as Google Gemma, LLama, Mistral, etc. To run chat completion, users can use the OpenAI Python library, `llma` CLI, or API endpoint.
weight: 1
---

## Chat Completion

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

You can also just call `client = OpenAI()` if you set environment variables `OPENAI_BASE_URL` and `OPENAI_API_KEY`.

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

## Tool Calling

vLLM requires additional flags ([link](https://docs.vllm.ai/en/latest/features/tool_calling.html) to use tool calling.
You can specify the flags with `vllmExtraFlags`. Here is an example configuration:

```yaml
inference-manager-engine:
  ...
  model:
    overrides:
      meta-llama-Meta-Llama-3.3-70B-Instruct-fp8-dynamic:
        runtimeName: vllm
        resources:
          limits:
            nvidia.com/gpu: 4
        vllmExtraFlags:
        - --chat-template
        - examples/tool_chat_template_llama3.1_json.jinja
        - --enable-auto-tool-choice
        - --tool-call-parser
        - llama3_json
        - --max-model-len
        - "8192"
```

Here is an example `curl` command:

```bash
curl \
  --request POST \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
  "model": "meta-llama-Meta-Llama-3.3-70B-Instruct-fp8-dynamic",
  "messages": [{"role": "user", "content": "What is the weather like in San Francisco?"}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "Get current weather for a given location.",
        "parameters": {
          "type": "object",
          "properties": {
          "location": {
            "type": "string",
            "description": "City and country"
          }
        }
      },
      "strict": true
    }
  }]
}' http://localhost:8080/v1/chat/completions
```

The output will have the `tool_calls` in its `message`.

```json
{
  ...
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "chatcmpl-tool-e698e3e36f354d089302b79486e4a702",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\": \"San Francisco, USA\"}"
            }
          }
        ]
      },
      "logprobs": null,
      "finish_reason": "tool_calls",
      "stop_reason": 128008
    }
  ],
  ...
}
```

## Audio Transcription

LLMariner supports the `/v1/audio/transcriptions` API. You can use a model like OpenAI/Whispter for this API.

CLI:

``` bash
llma audio transcriptions create --model openai-whisper-large-v3-turbo --file <audio file>
```

Python:

``` python
from openai import OpenAI

client = OpenAI(
  base_url="<Base URL (e.g., http://localhost:8080/v1)>",
  api_key="<API key secret>"
)

response = client.audio.transcriptions.create(
    model="openai-whisper-large-v3-turbo",
    file=open("<audio file>", "rb")
)
print(response)
```

curl:

``` bash
curl \
  --request POST \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header "Content-Type: multipart/form-data" \
  -F model=openai-whisper-large-v3-turbo \
  -F file="@<audio file>" \
  http://localhost:8080/v1/audio/transcriptions
```

## Model Response API

LLMariner supports the `/v1/responses` API. You can, for example, use `openai/gpt-oss-120b` for this API.

```bash
curl \
  --request POST \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header 'Content-Type: application/json' \
  --data '{
   "model": "openai-gpt-oss-120b",
   "input": "What is the capital of France?",
   "tools": [{
     "type": "function",
     "name": "get_weather",
     "description": "Get current temperature for a given location.",
     "parameters": {
       "type": "object",
       "properties": {
         "location": {
           "type": "string",
           "description": "City and country e.g. Bogotá, Colombia"
         }
       },
       "required": [
         "location"
       ],
       "additionalProperties": false
     }
   }],
   "tool_choice": "auto"
}' \
  http://localhost:8080/v1/responses
```

## Model Runtime Configuration

We currently support vLLM, Ollama, and Nvidia Triton Inference Server
as an inference runtime. You can change a runtime for each model. For
example, in the following configuration, the default runtime is set to
vLLM, and Ollama is used for `deepseek-r1:1.5b`.


```yaml
inference-manager-engine:
  ...
  model:
    default:
      runtimeName: vllm  # Default runtime
      resources:
        limits:
          nvidia.com/gpu: 1
    overrides:
      lmstudio-community/phi-4-GGUF/phi-4-Q6_K.gguf:
        preloaded: true
        vllmExtraFlags:
        - --tokenizer
        - microsoft/phi-4
      deepseek-r1:1.5b:
        runtimeName: ollama # Override the default runtime
        preloaded: true
        resources:
          limits:
            nvidia.com/gpu: 0
```

By default, one Pod serves only one pod. If you want to make one Ollama pod serve multiple models, you can set `dynamicModelLoading` to `true`.

```yaml
inference-manager-engine:
  ollama:
    dynamicModelLoading: true
```
