---
layout: single
title:  "My experience installing Ollama and Open WebUI"
categories: [ai]
tags: [kubernetes, ai]
excerpt: "Ugh, installing Ollama and Open Web UI locally (not with containers) was a pain, so here's why I don't prefer it" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
#### Background
I'm preparing to give a talk on Kubernetes and AI at this event: https://hottiesdodata.wordpress.com/

I'm not an AI expert, but I do know Kubernetes fairly well. One of my reasons for presenting is to learn from other presenters about the basics of AI.

I have just learned that my talk is only about an hour, so my plan is this:
Title: "Scaling AI with Kubernetes"
Agenda:
1. What is Kubernetes?
2. Why AI on Kubernetes?
3. Platform and security
4. Demo and Lab
5. Discussion

I suppose my talk will focus on the **why** and **how** of running AI on Kubernetes. The question I'm more scared of getting wrong is the fundamental **why** Kubernetes is the platform of choice. After all, any question of **how** is simply a matter of engineering, time, patience, research, etc.

#### Why Kubernetes?
I think I will start with "why containers". Here's why: I just tried to install Ollama and Open WebUI manually (not using containers) and hosed my own machine. I have now lost my Ubuntu VM that I use daily. I just can't get to it over the console (I run Windows 10 with Hyper-V and an Ubuntuy 22.04 VM). 

I am pretty well-behaved: I don't keep anything that I cannot lose on the VM, and I try to make everything as ephemeral as possible, but there's always *some* settings that are local, right? My AWS keys, my SSH private key, these things I have backed up. But - just - argh! It's still annoying when you lose a VM.

##### Why is this painful to do locally, without containers?
Here's my documenting how to set up these 2 things manually on a newly-deployed Ubuntu 22.04 VM:

###### Ollama
1. Install Ollama
  - [Download and install](https://ollama.com/download/linux) with one command: `curl -fsSL https://ollama.com/install.sh | sh`
  - That's it! Now check Ollama is installed: `curl http://localhost:11434`

###### Open WebUI
1.  First we need Node.js >= 20.10
  - [Instructions for Node.js](https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-20-04): `sudo apt update && sudo apt install nodejs` && `sudo apt install npm`
  - ` node -v && npm -v`
2. Now we need Python >= 3.11
  - [Instructions for Python](https://phoenixnap.com/kb/how-to-install-python-3-ubuntu): `sudo apt update && sudo apt install python3`
  - But `python3 --version` shows 3.10.12 
  - `sudo apt-get -y purge python3.10` <--- after this, still showing 3.10.12 is installed
  - `sudo apt autoremove` <--- after this, it appears python3 is uninstalled 
3. Let's try to install from PPA called Deadsnakes, a 3rd-party package repo
  -  i. `sudo apt update`
  -  ii. `sudo apt install software-properties-common && sudo apt update` <--- this actually reinstalls python3.10, but it's required because I need it to be able to use the `add-apt-repository` utility in the next command
  -  iii. `sudo add-apt-repository ppa:deadsnakes/ppa`
  -  iv. `sudo apt update`
  -  v. `sudo apt install python3.12` 
  -  vi. But `python3 --version` shows 3.10.12 ! Argh!! Now
  -  vii. `sudo apt-get remove python3.10`
  -  viii.`sudo apt-get autoremove`
  -  iv. But `python3.12 --version` now shows python 3.12 installed
    
Long story short - ARGH! - I am no expert but I'm getting stuck in local dependencies. I blew away the VM and deployed an Ubuntu 23.04 server VM, which should have Python 3.11 already installed. Then I re-installed Ollama, NodeJS and NPM. Then I followed the [instructions](https://docs.openwebui.com/getting-started/#install-from-open-webui-github-repo) to build Open WebUI from source.

This still failed!

I blew away 23.04 and redeployed 22.04 and ran `sudo snap install open-webui --beta` following [this advice](https://snapcraft.io/install/open-webui/ubuntu).

#### Let's verify Ollama and Open WebUI
1. Let's pull a model in Ollama
```bash
curl http://localhost:11434/api/pull -d '{
  "name": "llama3"
}'
```

2. Now let's make an API call to generate an answer for us:
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3",
  "prompt": "Why is the sky blue?"
}'
```
3. Interesting, we see that we get back a stream. A chunked HTTP response. Let's request it all delivered at once:
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

Looks like Ollama is working. Let's check Open Web UI.

```bash
curl http://localhost:8080
```

Yep, this looks reachable. I'll leave it here for now and update in another post.