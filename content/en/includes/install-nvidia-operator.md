---
draft: true
---
Nvidia GPU Operator is required to install the device plugin and make GPU resources visible in the K8s cluster. Run:

``` bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm upgrade --install --wait \
  --namespace nvidia \
  --create-namespace \
  gpu-operator nvidia/gpu-operator \
  --set cdi.enabled=true \
  --set driver.enabled=false \
  --set toolkit.enabled=false
```
