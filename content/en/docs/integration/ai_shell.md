---
title: AI Shell
description: Integrate with AI Shell to power your shell with the AI assistant.
weight: 6
---

[AI Shell](https://github.com/BuilderIO/ai-shell) is an open source tool that converts natural language to shell commands.

```bash
npm install -g @builder.io/ai-shell
ai config set OPENAI_API_ENDPOINT=<Base URL (e.g., http://localhost:8080/v1)>
ai config set OPENAI_KEY=<API key>
ai config set MODEL=<model name>
```

Then you can run the `ai` command and ask what you want in plain English and generate a shell command with a human readable explanation of it.

```bash
ai what is my ip address
```
