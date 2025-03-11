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

When you use a GGUF model with vLLM, please specify `--tokenizer=<original model>` in `vllmExtraFlags`. Here is an example
configuration for Phi 4.

```yaml
inference-manager-engine:
  ...
  model:
    default:
      runtimeName: vllm
    overrides:
      lmstudio-community/phi-4-GGUF/phi-4-Q6_K.gguf:
        preloaded: true
        resources:
          limits:
            nvidia.com/gpu: 1
        vllmExtraFlags:
        - --tokenizer
        - microsoft/phi-4
```

## Downloading model from your own S3 bucket

If you want to download models from your S3 bucket, you can specify the bucket configuration under
`model-manager-loader.downloader.s3`. For example, if you store model files under `s3://my-models/v1/base-models/<model-name>`,
you can specify the downloader config as follows:

```yaml
model-manager-loader:
  downloader:
    kind: s3
    s3:
      # The s3 endpoint URL.
      endpointUrl: https://s3.us-west-2.amazonaws.com
      # The region name where the models are stored.
      region: us-west-2
      # The bucket name where the models are stored.
      bucket: my-models
      # The path prefix of the model.
      pathPrefix: v1/base-models
```
