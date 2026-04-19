---
layout: single
title:  "Installing Coder for ephemeral dev environments"
categories: [ai]
tags: [coder, aws]
excerpt: "How to set up Coder quickly for a lab environment" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/coder/coder-logo.png"><img src="/assets/coder/coder-logo.png"></a>
    <figcaption></figcaption>
</figure>

[Coder](https://coder.com) is an open-source platform for self-hosted dev environments. It uses Terraform templates to define workspaces, which means your dev environment is code. You can version it, review it, and destroy it with confidence.

## Why ephemeral environments matter for Agentic AI
Agentic AI is different from regular AI-assisted coding. When an agent runs autonomously — executing shell commands, calling APIs, modifying files, spinning up processes — it needs an environment it can *own*. Shared dev environments are a liability. A runaway agent command, a dependency conflict, a leaked credential in a log file: these become everyone's problem in a shared space.

Ephemeral environments solve this. Spin one up, let the agent do its work, tear it down. No residue. No blast radius. This is the same logic that drove us to containerize workloads and adopt immutable infrastructure — and it applies even more strongly when the "developer" is an AI.

There's another angle here too: **enterprise compliance**. When you're operating in a large organization, you don't just need isolation for safety reasons. You need auditability, access control, and the ability to restrict *who* can reach *which* environment. That's where F5 BIG-IP APM comes in.

In this post I'm documenting my setup: Coder for ephemeral workspace management, running on a single EC2 instance to begin with.

---

## Deploy AWS EC2 instance, install Coder, configure basics
Coder can run on different platforms, including K8s. I'm going to use Ubuntu 24.04 on EC2 because it will be a quick lab. I'll use a `m7i.xlarge` instance and give myself 100 GB of disk space.

#### Install Docker
First, I have created a file at `/etc/docker/daemon.json` with this content:
```json
{
  "bip": "10.10.0.1/24"
}
```

This is because I want to choose the default bridge network CIDR block for this demo. Why? I am going to use a pre-configured IP range for my containers. This might come in handy later if I decide to implement a VPN for users to reach their docker containers. 

After this file exists, I install Docker on the Coder server. I use [my own instructions]({% post_url 2025-04-15-official-vs-unofficial-docker-packages %}).

#### Install Coder
Now install Coder:

```bash
# 1. Install Coder
curl -L https://coder.com/install.sh | sh

#Run this command to allow the binary to open raw sockets without requiring root:
sudo setcap cap_net_raw+ep $(which coder)

```
Notice that a user was created called `coder`:
```bash
sudo more /etc/passwd | grep coder
```

Notice these files that were created:
- `/usr/lib/systemd/system/coder.service`. - this defines the service and references an EnvironmentFile
- `/etc/coder.d/coder.env` - this EnvironmentFile holds configuration for the service

Notice that the files above exist, but the service is disabled:

```bash
sudo systemctl list-unit-files --state=disabled #Show disabled services
sudo systemctl status coder #this will show the daemon is not enabled
```

Edit the file `/etc/coder.d/coder.env` and add/edit these values:

```ini
CODER_ACCESS_URL=https://coder.my-f5.com
CODER_HTTP_ADDRESS='0.0.0.0:3000'
```
Also, add coder user to docker group, which will allow coder to run docker commands without sudo:

```bash
sudo usermod -aG docker coder
newgrp docker
```

Enable and start the coder service:
````bash
sudo systemctl enable coder
sudo systemctl start coder
````

I have a URL `coder.my-f5.com` that I have pointed at a typical VirtualServer on BIG-IP. It listens on HTTPS (tcp/443) and decrypts traffic with a clientSSL profile, but passes traffic to the Coder server on HTTP (tcp/3000) with **no** serverSSL profile.

In the next post, I'll offload authentication from Coder to BIG-IP APM.
