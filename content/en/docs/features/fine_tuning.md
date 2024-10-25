---
title: Model Fine-tuning
linkTitle: Fine-tuning
description: >
  This page describes how to fine-tune models with LLMariner.
weight: 4
---

## Submitting a Fine-Tuning Job

You can use the OpenAI Python library to submit a fine-tuning job. Here is an example snippet that uploads a training file and uses that to run a fine-tuning job.

``` python
from openai import OpenAI

client = OpenAI(
  base_url="<LLMariner Endpoint URL>",
  api_key="<LLMariner API key>"
)

file = client.files.create(
  file=open(training_filename, "rb"),
  purpose="fine-tune",
)

job = client.fine_tuning.jobs.create(
  model="google-gemma-2b-it",
  suffix="fine-tuning",
  training_file=file.id,
)
print('Created job. ID=%s' % job.id)
```

Once a fine-tuning job is submitted, a k8s Job is created. A Job runs in a namespace where a user\'s project is associated.

You can check the status of the job with the Python script or the `llma` CLI.

``` python
print(client.fine_tuning.jobs.list())
```

``` bash
llma fine-tuning jobs list
llma fine-tuning jobs get <job-id>
```

Once the job completes, you can check the generated models.

``` python
fine_tuned_model = client.fine_tuning.jobs.list().data[0].fine_tuned_model
print(fine_tuned_model)
```

Then you can get the model ID and use that for the chat completion request.

``` python
completion = client.chat.completions.create(
  model=fine_tuned_model,
  ...
```

## Debugging a Fine-Tuning Job

You can use the `llma` CLI to check the logs and exec into the pod.

``` bash
llma fine-tuning jobs logs <job-id>
llma fine-tuning jobs exec <job-id>
```

## Managing Quota

LLMariner allows users to manage GPU quotas with integration with [Kueue](https://kueue.sigs.k8s.io/).

You can install Kueue with the following command:

``` bash
export VERSION=v0.6.2
kubectl apply -f https://github.com/kubernetes-sigs/kueue/releases/download/$VERSION/manifests.yaml
```

Once the install completes, you should see `kueue-controller-manager` in the `kueue-system` namespace.

``` bash
$ kubectl get po -n kueue-system
NAME                                        READY   STATUS    RESTARTS   AGE
kueue-controller-manager-568995d897-bzxg6   2/2     Running   0          161m
```

You can then define `ResourceFlavor`, `ClusterQueue`, and `LocalQueue` to manage quota. For example, when you want to allocate 10 GPUs to `team-a` whose project namespace is `team-a-ns`, you can define `ClusterQueue` and `LocalQueue` as follows:

``` yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: team-a
spec:
  namespaceSelector: {} # match all.
  cohort: org-x
  resourceGroups:
  - coveredResources: [gpu]
    flavors:
    - name: gpu-flavor
      resources:
      - name: gpu
        nominalQuota: 10
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: team-a-ns
  name: team-a-queue
spec:
  clusterQueue: team-a
```
