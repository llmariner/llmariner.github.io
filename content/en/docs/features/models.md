---
title: Model Loading
linkTitle: Model Loading
description: >
  The following shows how to load models in LLMariner.
weight: 2
---

## Overview

LLMariner hosts LLMs in a Kubernetes cluster by downloading models from source repos and uploading to an S3-compatible object store. The supported source model repositories are following:

- LLMariner official model repository
- Hugging Face repositories
- Ollama repositories
- S3 bucket

If you already know models that you would like to download, you can specify them
in `values.yaml`. Here is an example configuration where two models are downloaded from
the LLMariner official model repository.

```yaml
model-manager-loader:
  baseModels:
  - google/gemma-2b-it-q4_0
  - sentence-transformers/all-MiniLM-L6-v2-f16
```

You can always update `values.yaml` and upgrade the Helm chart to download additional models.

You can also run `llma models create` to download additional models. For example, the following command
will download `deepseek-r1:1.5b` from the Ollama repository.

```bash
llma models create deepseek-r1:1.5b --source-repository ollama
```

You can check the status of the download with:

```bash
llma models list
```


{{% alert title="Note" color="primary" %}}
To download models from Hugging Face, you need additional configuration to embed the Hugging Face API key
to `model-manager-loader`. Please see the page below for details.
{{% /alert %}}

## Official Model Repository

This is the default configuration.
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
meta-llama/Llama-3.2-1B-Instruct             | None                       |  vLLM
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

## Hugging Face Repositories

First, create a k8s secret that contains the Hugging Face API key.

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

## Ollama Repositories

You can configure Ollama as model source repos by setting `model-manager-loader.downloader.kind` to `ollama`. The following is an example `values.yaml` that downloads `deepseek-r1:1.5b` from Ollama.

```yaml
model-manager-loader:
  downloader:
    kind: ollama
  baseModels:
  - deepseek-r1:1.5b
```


## S3 Bucket

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
      # Set to true if the bucket is public and we don't want to
      # use the credential attached to the pod.
      isPublic: false
```
