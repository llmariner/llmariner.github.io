---
title: Tutorials
description: >
  Here are the tutorials of LLMariner.
weight: 20
---

## Use OpenAI Python Library

You can download the following Jupyter Notebook and exercise the tutorial:

- <https://github.com/llmariner/llmariner/blob/main/tutorials/getting_started.ipynb>


## Fine-tune Models

This section goes through model fine-tuning using Llama3.2-1B-Instruct. We follow the following
page and create a model that answers questions in a sarcastic fashion.

- <https://medium.com/@alexandros_chariton/how-to-fine-tune-llama-3-2-instruct-on-your-own-data-a-detailed-guide-e5f522f397d7>

You can also check [this video](https://vimeo.com/1065288371) to see the overall flow.

### Step 1. Download a base model

Change the `model-manager-loader` config so that Llama3.2-1B-Instruct is downloaded to your S3 bucket.

```yaml
model-manager-loader:
  baseModels:
  - meta-llama/Llama-3.2-1B-Instruct
```

Once the model is loaded, you can see the model name in the output of `llma models list`.

### Step 2. Create a training dataset

Run the following Python script to create a training dataset file.

```python
from datasets import load_dataset
import json

dataset = load_dataset('csv', data_files={
    'train': 'https://raw.githubusercontent.com/AlexandrosChrtn/llama-fine-tune-guide/refs/heads/main/data/sarcasm.csv'
})['train']

with open('training.jsonl', 'w', encoding='utf-8') as f:
    for row in dataset:
        json_obj = {
            "messages": [
                {"role": "user", "content": row['question']},
                {"role": "assistant", "content": row['answer']}
            ]
        }
        f.write(json.dumps(json_obj) + '\n')

print("Training data has been saved to training.jsonl")
```

Verify that the first lines of the file looks like below:

```json
{"messages": [{"role": "user", "content": "Who invented the light bulb?"}, {"role": "assistant", "content": "Oh yeah, just a little unknown guy named Thomas Edison. You might have heard of him... if you pay attention at all."}]}
{"messages": [{"role": "user", "content": "Is the sky blue?"}, {"role": "assistant", "content": "Wow, you're asking that? Next, you'll tell me water is wet."}]}
```

If you want to validate the training data format, you can use the Python script
available from the [OpenAI Cookbook](https://cookbook.openai.com/examples/chat_finetuning_data_prep).

### Step 3. Upload a training dataset

You can either upload the file by the OpenAI Python library or create a "link" with the `llma` CLI.

If you use the OpenAI Python library, run the following script:

```python
from openai import OpenAI

client = OpenAI(
    base_url=os.getenv("LLMARINER_BASE_URL"),
    api_key=os.getenv("LLMARINER_API_KEY"),
)

file = client.files.create(
  file=open("./training.jsonl", "rb"),
  purpose="fine-tune",
)
print("Uploaded file. ID=%s" % file.id)
```

Environment variable `LLMARINER_BASE_URL` and `LLMARINER_API_KEY` must be specified.
You can generate an API key by running `llma auth api-keys create`.

You can also just create a file object by specifying the location of the S3 object path. For example,
the following command creates a new file object that points to `s3://<my-bucket>/training-data/training.jsonl`.

```bash
aws s3 cp training.jsonl s3://<my-bucket>/training-data/training.jsonl
llma storage files create-link \
  --object-path training-data/training.jsonl \
  --purpose fine-tune
```

The bucket (`<my-bucket>` in the above snippet) must be the same bucket
specified in the `global.objectStore.s3.bucke` field of Helm `values.yaml`.

### Step 4. Submit a fine-tuning job

Submit a job that fine-tunes Llama3.2-1B-Instruct. You can run the following Python
script by passing the file ID as its argument.

```python
import os
import sys

from openai import OpenAI

if len(sys.argv) != 2:
    print("Usage: python %s <training_file>" % sys.argv[0])
    sys.exit(1)

training_file = sys.argv[1]

client = OpenAI(
    base_url=os.getenv("LLMARINER_BASE_URL"),
    api_key=os.getenv("LLMARINER_API_KEY"),
)

resp = client.fine_tuning.jobs.create(
    model="meta-llama-Llama-3.2-1B-Instruct",
    suffix="fine-tuning",
    training_file=training_file,
    hyperparameters={
        "n_epochs": 20,
    }
)
print('Created job. ID=%s' % resp.id)
```

### Step 5. Wait for the job completion

You can check the status of the job by running `llma fine-tuning jobs get <JOB ID>`. You can
also check logs or exec into the container by running `llma fine-tuning jobs logs <JOB ID>`
or `llma fine-tuning jobs exec <JOB ID>`.


### Step 6. Run chat completion with the generated model.

Once the job completes, you can find the generated model by running `llma fine-tuning jobs get <JOB ID>` or `llma models list`.

Then you can use the model with chat completion. Example questions are:

- Who is Edison?
- Why is the sky blue?
