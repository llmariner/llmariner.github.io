---
draft: true
---

You can verify the installation by sending sample chat completion requests.

Note, if you have used LLMariner in other cases before you may need to delete the previous config by running `rm -rf ~/.config/llmariner`

``` bash
echo "This is your endpoint URL: ${INGRESS_CONTROLLER_URL}/v1"

llma auth login
# Type the above endpoint URL.

llma models list

llma chat completions create --model google-gemma-2b-it-q4_0 --role user --completion "what is k8s?"

llma chat completions create --model meta-llama-Meta-Llama-3.1-8B-Instruct-q4_0 --role user --completion "hello"
```
