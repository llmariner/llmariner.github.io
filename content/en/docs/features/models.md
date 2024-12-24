---
title: Supported Open Models
linkTitle: Supported Models
description: >
  The following shows the supported models.
weight: 2
---

## Models that are Officially Supported

The following is a list of supported models where we have validated.

Model                                        | Quantizations              |Supporting runtimes
---------------------------------------------|----------------------------|-------------------
TinyLlama/TinyLlama-1.1B-Chat-v1.0           | None                       |  vLLM
TinyLlama/TinyLlama-1.1B-Chat-v1.0           | AWQ                        |  vLLM
deepseek-ai/DeepSeek-Coder-V2-Lite-Base      | Q2_K, Q3_K_M, Q3_K_S, Q4_0 |  Ollama
deepseek-ai/DeepSeek-Coder-V2-Lite-Instruct  | Q2_K, Q3_K_M, Q3_K_S, Q4_0 |  Ollama
deepseek-ai/deepseek-coder-6.7b-base         | None                       |  vLLM, Ollama
deepseek-ai/deepseek-coder-6.7b-base         | AWQ                        |  vLLM
deepseek-ai/deepseek-coder-6.7b-base         | Q4_0                       |  vLLM, Ollama
fixie-ai/ultravox-v0_3                       | None                       |  vLLM
google/gemma-2b-it                           | None                       |  Ollama
google/gemma-2b-it                           | Q4_0                       |  Ollama
intfloat/e5-mistral-7b-instruct              | None                       |  vLLM
meta-llama/Meta-Llama-3.3-70B-Instruct       | AWQ, FP8-Dynamic           |  vLLM
meta-llama/Meta-Llama-3.1-70B-Instruct       | AWQ                        |  vLLM
meta-llama/Meta-Llama-3.1-70B-Instruct       | Q2_K, Q3_K_M, Q3_K_S, Q4_0 |  vLLM, Ollama
meta-llama/Meta-Llama-3.1-8B-Instruct        | None                       |  vLLM
meta-llama/Meta-Llama-3.1-8B-Instruct        | AWQ                        |  vLLM, Triton
meta-llama/Meta-Llama-3.1-8B-Instruct        | Q4_0                       |  vLLM, Ollama
nvidia/Llama-3.1-Nemotron-70B-Instruct       | Q2_K, Q3_K_M, Q3_K_S, Q4_0 |  vLLM
nvidia/Llama-3.1-Nemotron-70B-Instruct       | FP8-Dynamic                |  vLLM
mistralai/Mistral-7B-Instruct-v0.2           | Q4_0                       |  Ollama
sentence-transformers/all-MiniLM-L6-v2-f16   | None                       |  Ollama

Please note that some models work only with specific inference runtimes.

## Using Other Models in HuggingFace

First, create a k8s secret that contains the HuggingFace API key.

```bash
kubectl create secret generic \
  huggingface-key \
  -n llmariner \
  --from-literal=apiKey=${HUGGING_FACE_HUB_TOKEN}
```

The above command assumes that LLMarine runs in the `llmariner` namespace.

Then deploy LLMariner with the following `values.yaml`.

```yaml
model-manager-loader:
  downloader:
    kind: huggingFace
    huggingFace:
      cacheDir: /tmp/.cache/huggingface/hub
  huggingFaceSecret:
    name: huggingface-key
    apiKeyKey: apiKey

  baseModels:
  - Qwen/Qwen2-7B
  - TheBloke/TinyLlama-1.1B-Chat-v1.0-AWQ
```

Then the model should be loaded by `model-manager-loader`. Once the loading completes, the model name
should show up in the output of `llma models list`.
