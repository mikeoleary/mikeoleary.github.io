---
layout: single
title:  "My experience installing Ollama and Open WebUI - part 2"
categories: [ai]
tags: [kubernetes, ai]
excerpt: "Ugh, installing Ollama and Open Web UI locally (not with containers) was a pain, so here's why I don't prefer it" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
#### Background
After my last post, I have a script that will set up Ollama and Open WebUI manually (not using containers) on the Azure VM image "Ubuntu 23.10"

This is the machine type I start with:
```bash
"publisher": "canonical"
"sku": "23_10-gen2"
"version": "latest"
"exactVersion": "23.10.202405140"
```

These are the commands I run:

```bash
curl -fsSL https://ollama.com/install.sh | sh

sudo apt update
sudo apt install nodejs -y
sudo apt install npm -y
node -v && npm -v


sudo apt update
sudo apt install python3-pip -y
sudo mv /usr/lib/python3.11/EXTERNALLY-MANAGED /usr/lib/python3.11/EXTERNALLY-MANAGED.old

# pip install open-webui #this took around 5 mins to complete!

git clone https://github.com/open-webui/open-webui.git
cd open-webui/

# Copying required .env file
cp -RPp .env.example .env

# Building Frontend Using Node
npm i
npm run build

# Serving Frontend with the Backend
cd ./backend
pip install -r requirements.txt -U
python3 -m uvicorn main:app --reload
```

I am pretty sure I am committing bad practices here. This is just to document how I did it. If I have learned anything, it's the reminder of the headaches that are removed by using containers. Next post I may just document how to quickly do this with K8s.