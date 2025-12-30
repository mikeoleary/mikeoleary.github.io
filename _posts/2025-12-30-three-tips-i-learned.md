---
title: "3 handy tips I learned today"
date: 2025-12-30
categories: [ai]
tags: [kubernetes]
excerpt: "3 tips to come in handy someday"
toc: true
---

<figure>
    <a href="/assets/3tips/multi-tool-header-image.jpg"><img src="/assets/3tips/multi-tool-header-image.jpg"></a>
    <figcaption>No matter how senior you get, you can always use tools for your toolbag</figcaption>
</figure>


I try to avoid science projects. However, I'll admit that there are many side-benefits to individual hands-on projects. Soon I will write about what I consider to be the difference between a "science project" and a valuable individual exercise for a Solution Architect to undertake.

Today, while working on a solution that I'm writing about [starting here]({% post_url 2025-12-6-openwebui-plus-chatgpt %}), I had to do some troubleshooting and learned 3 new tips I thought to document:

### 3 tips I learned today

1. **Restart Chrome and keep your open tabs**<br>
  Type `chrome://restart` in the browser bar. Watch Chrome restart and your open tabs are re-launched for you. Use this for troubleshooting (eg, force new TLS handshake, switch mTLS certs, etc). This if for when you don't want to use incognito mode or another Chrome profile.

2. **Ephemeral debug containers**<br>
  I don't know why I didn't know about [ephemeral debug containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) earlier. Often I was using a busybox or ubuntu pod that I would spin up manually for troubleshooting. I even wrote a [post]({% post_url 2023-09-05-easy-kubectl-to-remember %}) including an example. Anyway, these commands allowed me to:
    - run an ephemeral debug container with image of ubuntu, sharing the network and IPC namespace of the target container
    - install tcpdump on this container
    - run tcpdump in NGINX Ingress Controller even though tcpdump is not included with that container image:
    ```bash
    #run a debug container in existing pod and target existing container
    kubectl debug -it nginx-ingress-748f7c5df6-gjqdj -n nginx-ingress --image=ubuntu:latest --target=nginx-ingress --share-processes  -- /bin/bash
    #once you are at command line of your debug container, you can install troubleshooting tools
    apt update
    apt install tcpdump
    tcpdump port 8080 -w /tmp/pcap2.pcap
    ```
  *"In Kubernetes ephemeral debugging, the `--target` flag specifies the **name of an existing container within the same pod** that your new debugger container should share its process (PID), network, and IPC namespaces with, enabling deep inspection of its processes and filesystem without restarting the pod. It's crucial for multi-container pods to direct the debugging tools (like bash, curl, strace) to the correct application container for troubleshooting, especially when using minimal images (like distroless) that lack debug tools."*

3. **How to copy files from a container quickly**<br>
  I can't believe I haven't done this before. Probably I have and just don't remember. Often I am doing something similar with different methods (eg. mounting a file via a ConfigMap, or running an exec command and piping the value to cat a new file). Anyway, here's the command I ran to copy a file from my container to my workstation running kubectl.
```bash
#Note: debugger-98hqg was the name given to my ephemeral debug container. I learned this from the output of my kubectl debug command.
#Note2: nginx-ingress-748f7c5df6-gjqdj was the name of my NGINX Ingress Controller pod
kubectl cp -n nginx-ingress nginx-ingress-748f7c5df6-gjqdj:/tmp/pcap2.pcap /tmp/pcap2.pcap -c debugger-98hqg
```
