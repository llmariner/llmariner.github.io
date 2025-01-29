---
title: Weights & Biases (W&B)
linkTitle: Weights & Biases
description: Integration with W&B and see the progress of your fine-tuning jobs.
weight: 11
---

[Weights and Biases (W&B)](https://wandb.ai/) is an AI developer platform. LLMariner provides the integration with W&B so that metrics for fine-tuning jobs are reported to W&B. With the integration, you can easily see the progress of your fine-tuning jobs, such as training epoch, loss, etc.

Please take the following steps to enable the integration.

First, obtain the API key of W&B and create a Kubernetes secret.

``` bash
kubectl create secret generic wandb
  -n <fine-tuning job namespace> \
  --from-literal=apiKey=${WANDB_API_KEY}
```

The secret needs to be created in a namespace where fine-tuning jobs run. Individual projects specify namespaces for fine-tuning jobs, and the default project runs fine-tuning jobs in the \"default\" namespace.

Then you can enable the integration by adding the following to your Helm `values.yaml` and re-deploying LLMariner.

``` yaml
job-manager-dispatcher:
  job:
    wandbApiKeySecret:
      name: wandb
      key: apiKey
```

A fine-tuning job will report to W&B when the `integration` parameter is specified.

``` python
job = client.fine_tuning.jobs.create(
  model="google-gemma-2b-it",
  suffix="fine-tuning",
  training_file=tfile.id,
  validation_file=vfile.id,
  integrations=[
    {
      "type": "wandb",
      "wandb": {
         "project": "my-test-project",
      },
    },
  ],
)
```

Here is an example screenshot. You can see metrics like `train/loss` in the W&B dashboard.

{{< img src="/images/wandb.png" >}}

{{% alert title="Note" color="primary" %}}
The W&B API key is attached to fine-tuning jobs as an environment variable. Please be careful of its visibility.
{{% /alert %}}
