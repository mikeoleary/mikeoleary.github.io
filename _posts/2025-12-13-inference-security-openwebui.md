---
title: "Inference security using Open WebUI"
date: 2025-12-12
categories: [ai]
tags: [f5, ai, guardrails, openwebui, kubernetes]
excerpt: "A basic intro. How to extend Open Web UI to add a layer of inference security"
toc: true
---

|AI inference networking |
|-------|--------|---------|
| Part 1 | [Open WebUI as front door for ChatGPT]({% post_url 2025-12-6-openwebui-plus-chatgpt %}) |
| Part 2 | [Inference security using Open WebUI]({% post_url 2025-12-13-inference-security-openwebui %}) |
| Part 3 | [Layering NGINX for more Inference security]({% post_url 2025-12-23-nginx-openwebui %}) |


<figure>
    <a href="/assets/openwebui-chatgpt/ai-inference-demo-header_2.svg"><img src="/assets/openwebui-chatgpt/ai-inference-demo-header_2.svg"></a>
    <figcaption>Adding inference security using F5 AI Guardrails.</figcaption>
</figure>

### How to add guardrails using Open WebUI
In the previous post we covered 3 easy reasons to use Open WebUI:
- chat history kept on-prem
- settings kept on-prem
- drastically cheaper with pay-as-you-go API usage

Now we will explore more extensibility with Open WebUI [functions](https://docs.openwebui.com/features/plugin/functions/). In this article we will show how to add a 3rd party prompt scanner (F5 AI Guardrails). We could use Functions for other things, like
- adding mTLS verification of clients
- directing some users' prompts to one LLM, and others' to another LLM
- chaining together multiple LLM's
- etc

Perhaps in future. For now, here's how I added F5 AI Guardrails to my Open WebUI config

#### Pre-requisites
- A tenant in CalypsoAI SaaS environment with a Project configured and at least 1 active scanner for testing blocked prompts.
- A project API token from CalypsoAI.

#### Instructions
1. Deploy Open WebUI. I use K8s, so follow the previous post.
2. In addition, this time create a secret in K8s to hold our API key for F5 AI Guardrails. Since this was called CalypsoAI before F5 acquired the tech, we'll call the secret calypso-api-key:
```bash
kubectl create secret generic calypso-api-key --from-literal=CALYPSO_API_KEY=your_real_api_key --namespace open-webui
```
3. We'll use this as an env var, so add this to your YAML file for the Open WebUI deployment:
```yaml
# ...
        env: # existing line
        - name: OLLAMA_BASE_URL  # existing line
          value: "http://ollama-service.open-webui.svc.cluster.local:11434"  # existing line
        - name: CALYPSO_API_KEY
          valueFrom:
            secretKeyRef:
              name: calypso-api-key
              key: CALYPSO_API_KEY
# ...
```
4. Now deploy Open WebUI with K8s, expose it, and log in. Don't add your API token for OpenAI in OpenWebUI (we'll add it to F5 AI Guardrails which is in-line)
5. Copy the function below into Open WebUI.
    <figure>
      <a href="/assets/openwebui-chatgpt/open-webui-screenshot1.png"><img src="/assets/openwebui-chatgpt/open-webui-screenshot1.png"></a>
      <figcaption>Under 'Admin Panel' you will find Functions</figcaption>
    </figure>

    <figure>
      <a href="/assets/openwebui-chatgpt/open-webui-screenshot2.png"><img src="/assets/openwebui-chatgpt/open-webui-screenshot2.png"></a>
      <figcaption>Copy the function below, give it a name and description.</figcaption>
    </figure>

    <figure>
      <a href="/assets/openwebui-chatgpt/open-webui-screenshot3.png"><img src="/assets/openwebui-chatgpt/open-webui-screenshot3.png"></a>
      <figcaption>Don't forget to enable the function</figcaption>
    </figure>

6. Notice a few things:
  - we do not hardcode our Calypso API key. This variable can be added as a "valve" (like a variable), or the default will be the env var we created earlier.
  - we have hardcoded a few other variables, like `Model Name`, `Base Url`, and a few others. Feel free to edit these variables.
7. Now you should be able to return to the chat interface and send a prompt to CalypsoAI (F5 AI Guardrails).

    <figure>
      <a href="/assets/openwebui-chatgpt/open-webui-screenshot4.png"><img src="/assets/openwebui-chatgpt/open-webui-screenshot4.png"></a>
      <figcaption>Function is sending our prompt to Calypso. A policy disallowing any discussion of basketball is blocking our prompt.</figcaption>
    </figure>

8. Notice that if your prompt is blocked by Calypso, you can see the name of the scanner that blocked it. This level of detail must be enabled with a valve.

I know this is short and sweet. You need a CalypsoAI tenant and API key, and those are not given away. Only F5'ers are likely to use this post, so please reach out if you want help.

#### Open WebUI function
<script src="https://gist.github.com/mikeoleary/56e78722564d43547976a1f35841f2b0.js"></script>
