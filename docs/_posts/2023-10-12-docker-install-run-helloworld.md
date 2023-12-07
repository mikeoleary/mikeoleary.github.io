---
layout: post
title:  "Docker install and run hello world"
date:   2023-10-25 09:00:00 -0400
categories: Docker
---

This is purely for my own note taking. 

I run these commands a few times a month when I'm setting up a VM for troubleshooting purposes. Hopefully I can find this blog post faster than I can search my Notepad++ tabs.

### Ubuntu bash commands to install Docker and run a container

{% highlight bash %}
sudo apt update
sudo apt install docker.io -y
 
# start and enable service
sudo systemctl start docker
sudo systemctl enable docker
 
# to check and see if Ubuntu is running the docker service, uncomment below.
#systemctl status docker
 
# Add your user to Docker group to avoid typing sudo everytime you run docker commands.
sudo usermod -aG docker ${USER}
newgrp docker

docker run -d -p 80:8080 -p 443:8443 --restart unless-stopped f5devcentral/f5-hello-world
#docker run -d -p 80:8080 --restart unless-stopped nginxdemos/nginx-hello
{% endhighlight %}




