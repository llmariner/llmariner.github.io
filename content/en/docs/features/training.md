---
title: General-purpose Training
description: >
  LLMariner allows users to run general-purpose training jobs in their Kubernetes clusters.
weight: 6
---

## Creating a Training Job

You can create a training job from the local pytorch code by running the following command.

``` bash
llma batch jobs create \
  --image="pytorch-2.1" \
  --from-file=my-pytorch-script.py \
  --from-file=requirements.txt \
  --file-id=<file-id> \
  --command "python -u /scripts/my-pytorch-script.py"
```

Once a training job is created, a k8s Job is created. The job runs the command specified in the `--command` flag, and files specified in the `--from-file` flag are mounted to the /scripts directory in the container. If you specify the `--file-id` flag (optional), the file will be download to the /data directory in the container.

You can check the status of the job by running the following command.

``` bash
llma batch jobs list
llma batch jobs get <job-id>
```

## Debugging a Training Job

You can use the `llma` CLI to check the logs of a training job.

``` bash
llma batch jobs logs <job-id>
```

## PyTorch Distributed Data Parallel

LLMariner supports PyTorch Distributed Data Parallel (DDP) training. You can run a DDP training job by specifying the number of per-node GPUs and the number of workers in the `--gpu` and `--workers` flags, respectively.

``` bash
llma batch jobs create \
  --image="pytorch-2.1" \
  --from-file=my-pytorch-ddp-script.py \
  --gpu=1 \
  --workers=3 \
  --command "python -u /scripts/my-pytorch-ddp-script.py"
```

Created training job is pre-configured some DDP environment variables; MASTER_ADDR, MASTER_PORT, WORLD_SIZE, and RANK.
