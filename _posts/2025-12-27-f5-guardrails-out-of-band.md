---
title: "F5 AI Guardrails out-of-band"
date: 2025-12-27
categories: [ai]
tags: [f5, ai, guardrails, openwebui, kubernetes]
excerpt: "Intro gets more interesting with out-of-band prompt security"
toc: true
---

|AI inference networking |
|-------|--------|---------|
| Part 1 | [Open WebUI as front door for ChatGPT]({% post_url 2025-12-6-openwebui-plus-chatgpt %}) |
| Part 2 | [Inference security using Open WebUI]({% post_url 2025-12-13-inference-security-openwebui %}) |
| Part 3 | [Layering NGINX for more Inference security]({% post_url 2025-12-23-nginx-openwebui %}) |
| Part 4 | [F5 AI Guardrails out-of-band]({% post_url 2025-12-27-f5-guardrails-out-of-band %}) |

<figure>
    <a href="/assets/openwebui-chatgpt/ai-inference-demo-header_4.svg"><img src="/assets/openwebui-chatgpt/ai-inference-demo-header_4.svg"></a>
    <figcaption>Moving to out-of-band model for prompt security</figcaption>
</figure>

### Why put AI guardrails out-of-band?
I'm aiming for a [model-agnostic approach](https://calypsoai.com/insights/the-power-of-a-model-agnostic-approach-in-ai-deployment/) in AI. Actually I'd like to be agnostic on almost everything. By running F5 AI Guardrails out-of-band, I plan to move some orchestration decisions back to my Open WebUI instance.

For example, I want to be able to do things like this:
- Multiple models
  - I may switch models easily from Open WebUI without touching F5 AI Guardrails
  - Perhaps if Guardrails flags a prompt or response, I'll have options at Open WebUI. Examples:
    - I could send a "blocked" prompt to an on-prem model
    - I could try my hand at redacting offending text from a flagged response
- Policy enforcement centralized
  - depending on *which* scanner blocks a prompt or a response, I may or may not honor the decision of Guardrails
  - I could bypass the Guardrails checks selectively
- etc

### Pre-requisites
- Follow blog posts 1, 2 & 3. By now you should be able to deploy and destroy our environment on demand.

### Function for out-of-band prompt/response inspection
<script src="https://gist.github.com/mikeoleary/7ba20d283caf8545ea9378b5696d8044.js"></script>

### Conclusion
