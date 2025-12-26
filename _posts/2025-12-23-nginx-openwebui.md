---
title: "Layering NGINX for more Inference security"
date: 2025-12-23
categories: [ai]
tags: [f5, ai, guardrails, openwebui, kubernetes]
excerpt: "Basic intro continues. Adding NGINX Ingress Controller to our architecture"
toc: true
---

|AI inference networking |
|-------|--------|---------|
| Part 1 | [Open WebUI as front door for ChatGPT]({% post_url 2025-12-6-openwebui-plus-chatgpt %}) |
| Part 2 | [Inference security using Open WebUI]({% post_url 2025-12-13-inference-security-openwebui %}) |
| Part 3 | [Layering NGINX for more Inference security]({% post_url 2025-12-23-nginx-openwebui %}) |

<figure>
    <a href="/assets/openwebui-chatgpt/ai-inference-demo-header_3.svg"><img src="/assets/openwebui-chatgpt/ai-inference-demo-header_3.svg"></a>
    <figcaption>Adding NGINX Ingress Controller for mTLS auth.</figcaption>
</figure>

### Why add NGINX to our architecture?
In the previous posts we covered Open WebUI and F5 AI Guardrails. Now we'll add NGINX ingress controller so that we can do some mTLS auth. After that we'll experiment with adding other functionality.

### Pre-requisites
- Follow blog posts 1 & 2. By now you should be able to deploy and destroy our environment on demand.

### Instructions to deploy NGINX Ingress Controller
1. Deploy Open WebUI + CalypsoAI integration on K8s following previous posts.
2. I've cloned the repo for [NGINX Ingress Controller](https://github.com/nginx/kubernetes-ingress) at version 5.3.1
  - I have added the relevant manifests to the [repo](https://github.com/mikeoleary/my-secure-ai-app/) accompanying this series.
  - The [instructions to deploy](https://github.com/mikeoleary/my-secure-ai-app/blob/300eeac1af73f10a3b8331070bcd055b8243f503/deploy_notes.md) now include all manifests in the correct order. (The linked instructions are a point-in-time version.)
3. Now we should have NGINX I.C. running. You should be able to access Open WebUI via public IP address, and your traffic should be traversing NGINX I.C.
  <figure>
    <a href="/assets/openwebui-chatgpt/screenshot-open-webui.png"><img src="/assets/openwebui-chatgpt/screenshot-open-webui.png"></a>
    <figcaption></figcaption>
  </figure>

#### Instructions to deploy cert-manager, server cert, and mTLS certs.
1. Install cert-manager via manifests:
  ```bash
  kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml
  ```
2. Create a ClusterIssuer
3. Create a CACert
4. Create a (namespaced) Issuer using the secret from the CACert
5. Create a secret of type `nginx.org/ca` where the only data field is "ca.crt" and the value is the certificate material. 
  ```bash
  # Extract the tls.crt from the ca-cert and create secret of type nginx.org/ca
  # this is so we have a secret in the correct format for NGINX's Policy CRD, which will implement mTLS.
  kubectl get secret open-webui-ca-cert -n open-webui -o json | jq '.type="nginx.org/ca" | .data={"ca.crt": .data."tls.crt"} | .metadata={"name": "open-webui-ca-cert-nginx", "namespace": "open-webui"}' | kubectl apply -f - 
  ```
6. Create a secret for TLS for NGINX, and multiple mTLS client certs


#### Instructions to configure NGINX Ingress Controller
1. Create a [Policy CR](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/#ingressmtls) that requires mTLS verification. [My example](https://github.com/mikeoleary/my-secure-ai-app/blob/d1740ebee9a6c08c4e31fdbe139356dc36471357/open-webui-nginx-config/policy.yaml).
2. Create a VirtualServer CR that implements TLS with our server cert and references our Policy impelemting mTLS. [My example](https://github.com/mikeoleary/my-secure-ai-app/blob/d1740ebee9a6c08c4e31fdbe139356dc36471357/open-webui-nginx-config/virtualserver.yaml).

#### Now, test your access to Open WebUI
1. Export CA Cert into trusted root store of your VM (unless using public CA)
2. Export mTLS client cert/key into your VM
3. Access the website, using a DNS name that matches your TLS cert (required by Chrome to prompt for mTLS certs)

  <figure>
    <a href="/assets/openwebui-chatgpt/open-webui-screenshot5.png"><img src="/assets/openwebui-chatgpt/open-webui-screenshot5.png"></a>
    <figcaption>If you can load the Open WebUI site only by providing a mTLS client cert, you have correctly configured NGINX to require and verify client certificates.</figcaption>
  </figure>

### Conclusion
We've added mTLS authentication to our environment, by using NGINX Ingress Controller. Next we'll use the identity learned via mTLS to make decisions at the time of inference within Open WebUI.