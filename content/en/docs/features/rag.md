---
title: Retrieval-Augmented Generation (RAG)
description: >
  This page describes how to use RAG with LLMariner.
---

## An Example Flow

The first step is to create a vector store and create files in the vector store. Here is an example script with the OpenAI Python library:

``` python
from openai import OpenAI

client = OpenAI(
  base_url="<LLMariner Endpoint URL>",
  api_key="<LLMariner API key>"
)

filename = "llmariner_overview.txt"
with open(filename, "w") as fp:
  fp.write("LLMariner builds a software stack that provides LLM as a service. It provides the OpenAI-compatible API.")
file = client.files.create(
  file=open(filename, "rb"),
  purpose="assistants",
)
print("Uploaded file. ID=%s" % file.id)

vs = client.beta.vector_stores.create(
  name='Test vector store',
)
print("Created vector store. ID=%s" % vs.id)

vfs = client.beta.vector_stores.files.create(
  vector_store_id=vs.id,
  file_id=file.id,
)
print("Created vector store file. ID=%s" % vfs.id)
```

Once the files are added into vector store, you can run the completion request with the RAG model.

``` python
from openai import OpenAI

client = OpenAI(
  base_url="<Base URL (e.g., http://localhost:8080/v1)>",
  api_key="<API key secret>"
)

completion = client.chat.completions.create(
  model="google-gemma-2b-it-q4_0",
  messages=[
    {"role": "user", "content": "What is LLMariner?"}
  ],
  tool_choice = {
   "choice": "auto",
   "type": "function",
   "function": {
     "name": "rag"
   }
 },
 tools = [
   {
     "type": "function",
     "function": {
       "name": "rag",
       "parameters": "{\"vector_store_name\":\"Test vector store\"}"
     }
   }
 ],
  stream=True
)
for response in completion:
  print(response.choices[0].delta.content, end="")
print("\n")
```

If you want to hit the API endpoint directly, you can use `curl`. Here is an example.

``` bash
curl \
  --request POST \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
   "model": "google-gemma-2b-it-q4_0",
   "messages": [{"role": "user", "content": "What is LLMariner?"}],
   "tool_choice": {
     "choice": "auto",
     "type": "function",
     "function": {
       "name": "rag"
     }
   },
   "tools": [{
     "type": "function",
     "function": {
     "name": "rag",
       "parameters": "{\"vector_store_name\":\"Test vector store\"}"
     }
 }]}' \
 http://localhost:8080/v1/chat/completions
```

## Embedding API

If you want to just generate embeddings, you can use the Embedding API, which is compatible with the OpenAI API.

Here are examples:

``` bash
llma embeddings create --model intfloat-e5-mistral-7b-instruct --input "sample text"


curl \
  --request POST \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
   "model": "sentence-transformers-all-MiniLM-L6-v2-f16",
   "input": ""sample text,
 }' \
 http://localhost:8080/v1/embeddings
```
