---
layout: single
title:  "Installing Docker using the official repo"
categories: [docker]
tags: [docker]
excerpt: "I used to install Docker as quickly as possible. These days I prefer to use the official repository from Docker" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/install-docker-official-repo/docker-official-unofficial.png"><img src="/assets/install-docker-official-repo/docker-official-unofficial.png"></a>
</figure>

### Summary
This is a quick script to copy to install Docker on Ubuntu from the official repository.

### Why am I writing this? 
In my job, I often deploy a Docker host, and for whatever reason I often start with Ubuntu and deploy Docker and a demo app. To make this super-quick, I wrote a [post]({% post_url 2023-10-12-docker-install-run-helloworld %}) that I end up referencing as a shortcut pretty often. 

Recently I've tried to be more deliberate and specific about versions, integrity, and sources of software. And this got me thinking:
- Docker is open source and there's different versions
- There's many ways to install on Ubuntu alone
- It's almost never a problem, but if someone asks me what version of Docker I'm running, I actually don't know!

### How do I like to install Docker now?
Docker can be installed manually using multiple methods and from various package managers that come with your OS
  - in the past, I have used the `docker.io` package using `apt-get` because I didn't know a better way 
  - now, I prefer to use [these instructions](https://docs.docker.com/engine/install/ubuntu/) because they come from Docker's website.
      
Of the recommended options, I prefer to use `apt-get` because it's easy, but I'd like to pull the package from Docker's repository for Apt. Here's a link to [that spot](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) in the doc.

### Install Docker from offical repo and run hello-world
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to Docker group to avoid typing sudo everytime you run docker commands.
sudo usermod -aG docker ${USER}
newgrp docker

docker run -d -p 80:8080 -p 443:8443 --restart unless-stopped f5devcentral/f5-hello-world
#docker run -d -p 80:8080 --restart unless-stopped nginxdemos/nginx-hello
```
    