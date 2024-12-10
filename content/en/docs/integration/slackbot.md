---
title: Slackbot
description: Build a Slackbot that integrates with LLMariner
weight: 7
---

You can build a Slackbot that is integrated with LLMariner. The bot can provide a chat UI with Slack
and answer questions from end users.

An example implementation can be found in https://github.com/llmariner/slackbot. You can deploy it in your
Kubernetes clusters and build a Slack app with the following configuration:

- Create an app-level token whose scope is `connections:write`.
- Enable the socket mode. Enable event subscription with the `app_mentions:read` scope.
- Add the following scopes in "OAuth & Permissions": `app_mentions:read`, `chat:write`, `chat:write.customize`, and `links:write`

You can install the Slack application to your workspace and interact.

{{< img src="/images/slackbot.png" >}}
