---
title: Installation
description: Choose the guide that best suits your needs and platform.
weight: 10
---

LLMariner takes ControlPlane-Worker model. The control plane gets a request and gives instructions to the worker while the worker processes a task such as inference.

Both components can operate within a single cluster, but if you want to utilize GPU resources across multiple clusters, they can also be installed into separate clusters.

<!-- original file is located at diagram/llmariner.excalidraw -->
{{< img src="images/install_modes.png" >}}
