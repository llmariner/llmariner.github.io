---
title: Supported Open Models
description: >
  The following shows the supported models.
weight: 2
---

Please note that some models work only with specific inference runtimes.

Model                                        | Quantizations              |Supporting runtimes
---------------------------------------------|----------------------------|---------------------
TinyLlama/TinyLlama-1.1B-Chat-v1.0           | None                       |  vLLM
TinyLlama/TinyLlama-1.1B-Chat-v1.0           | AWQ                        |  vLLM
deepseek-ai/DeepSeek-Coder-V2-Lite-Base      | Q2_K, Q3_K_M, Q3_K_S, Q4_0 |  Ollama
deepseek-ai/DeepSeek-Coder-V2-Lite-Instruct  | Q2_K, Q3_K_M, Q3_K_S, Q4_0 |  Ollama
deepseek-ai/deepseek-coder-6.7b-base         | None                       |  vLLM, Ollama
deepseek-ai/deepseek-coder-6.7b-base         | AWQ                        |  vLLM
deepseek-ai/deepseek-coder-6.7b-base         | Q4_0                       |  vLLM, Ollama
google/gemma-2b-it                           | None                       |  Ollama
google/gemma-2b-it                           | Q4_0                       |  Ollama
intfloat/e5-mistral-7b-instruct              | None                       |  vLLM
meta-llama/Meta-Llama-3.1-70B-Instruct       | AWQ                        |  vLLM
meta-llama/Meta-Llama-3.1-70B-Instruct       | Q2_K, Q3_K_M, Q3_K_S, Q4_0 |  vLLM, Ollama
meta-llama/Meta-Llama-3.1-8B-Instruct        | None                       |  vLLM
meta-llama/Meta-Llama-3.1-8B-Instruct        | Q4_0                       |  vLLM, Ollama
mistralai/Mistral-7B-Instruct-v0.2           | Q4_0                       |  Ollama
sentence-transformers/all-MiniLM-L6-v2-f16   | None                       |  Ollama
