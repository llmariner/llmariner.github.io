---
title: API and GPU Usage Optimization
linkTitle: API/GPU Usage Optimization
description:
weight: 7
---

{{% alert title="Note" color="primary" %}}
Work-in-progress.
{{% /alert %}}

## API Usage Visibility

## Inference Request Rate-limiting

## Optimize GPU Utilization

### Auto-scaling of Inference Runtimes

### Scheduled Scale Up and Down of Inference Runtimes

If you want to scale up/down model runtimes based on certain schedules, you can enable the scheduled shutdown feature.

Here is an example `values.yaml`.

```yaml
  inference-manager-engine:
    runtime:
      scheduledShutdown:
        enable: true
        schedule:
         # Pods are up between 9AM to 5PM.
         scaleUp:   "0 9 * * *"
         scaleDown: "0 17 * * *"
         timeZone: "Asia/Tokyo"
```
