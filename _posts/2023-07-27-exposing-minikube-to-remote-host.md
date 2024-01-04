---
layout: single
toc: true
title:  "Exposing apps in Minikube to remote hosts"
categories: [kubernetes]
tags: [kubernetes,linux,minikube]
excerpt: "Minikube is a handy tool for a quick K8s cluster running locally. But it's only available locally, not to other machines on the network. Here's a few ways to solve for this." #this is a custom variable meant for a short description to be displayed on home page
---
### Summary
I recently had a need to deploy [Minikube](https://minikube.sigs.k8s.io/docs/) on my Ubuntu VM. I was able to run a pod and test with a `curl` command from my Ubuntu host. But regular networking rules didn't quite explain why I couldn't access my application from a remote host. I ended up using `kubectl port-forward` to solve for this, but `socat` and NGINX are other options. I'll explain how I did this below.

![Minikube logo](/assets/accessing-minikube-remotely/minikube-logo-1024x290.jpg)

### The environment setup
My hardware is a Dell laptop running Windows 10. I have Hyper-V running, and my Ubuntu 20.04 machine is a VM. Running minikube on an Ubuntu VM is pretty straightforward.

This was my starting point:
- Ubuntu 20.04 VM on Hyper-V
- Docker CE version 20.10.21 was already installed
- kubectl was alreaady installed

Here's how to install Minikube on a Ubuntu VM when Docker is already installed:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --driver=docker --addons=ingress
```
### Exposing an app within Minikube

#### Deploying the app
Now that Minikube is running, let's deploy and expose an app. I've used the imperative kubectl commands here and deployed a `pod` (not a `deployment`) because I'm trying to keep this short and sweet:
```bash
kubectl run f5-hello-world --image=f5devcentral/f5-hello-world --replicas=1
kubectl expose pod f5-hello-world --port=8080 --type=NodePort
```
Great, now I can learn the NodePort service's high port and curl from the local Ubuntu host:
```bash
$ minikube ip # let's get the IP address of the minikube node
192.168.49.2
$ kubectl get node -o json | jq .items[].status.addresses[0].address -r # here's another way you could get that IP address
192.168.49.2
$ kubectl get svc  # here is the way to learn the high port of the NodePort service, if you didn't specify one yourself. In this case it's 31193
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
f5-hello-world   NodePort    10.111.51.254   <none>        8080:31193/TCP   2m13s
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          4m43s
$ curl 192.168.49.2:31193 ## Now let's curl the node IP address and high port
<!--
        f5-hello-world - Main Page
        https://github.com/f5devcentral/f5-hello-world
        Artiom Lichtenstein
        v2.2.0, 19/12/2017
-->
........... ## I've redacted the remainder of the http response for brevity.
```

#### Minikube is only accessible locally, by default
Great, I have a NodePort service that I can reach by hitting the one and only node. Now I should be able to reach that same IP address and port from another machine, as long as my Ubuntu VM is forwarding IP traffic and allowing the correct ports, right?

Let's edit `/etc/sysctl.conf` to enable packet forwarding

{% highlight bash %}
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1
{% endhighlight %}

Let's also use `ufw` to allow port 31193/TCP:
```bash
sudo ufw allow 31193
```

And let's allow IP tables to do IP masquerading, trying to turn our Ubuntu VM into a router. I can never remember these so here's the [link](https://www.opensourceforu.com/2015/04/how-to-configure-ubuntu-as-a-router/) I copied these from.
```bash
sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
sudo iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
```

Now, if I go to another host (like my Windows machine) and set a route for 192.168.49.0/24 where the next hop is my (directly connected) Linux VM, that should allow me to use a broswer and hit http://192.168.49.2:31193, right?

The answer is no. [By design](https://minikube.sigs.k8s.io/docs/faq/#how-can-i-access-a-minikube-cluster-from-a-remote-network), minikube is meant to only listen on the local network. If we want to hit this application from a machine other than our Ubuntu VM, we're gonig to need a solution to proxy traffic. 

### How you can expose the app to a remote network
I found three solutions, but more probably exist: kubectl port forwarding, a tool called socat, or NGINX (a web proxy).

#### kubectl port-forward
The fastest way to expose your minikube app is to use `kubectl port-forward`. This is a great way to get a port exposed on my Ubuntu host and have the traffic forwarded to a service or pod. I can then hit this exposed port on the Ubuntu VM from a remote host.

In the example below, 172.19.104.149 is my Ubuntu host.
```bash
$ kubectl port-forward --address=172.19.104.149 pod/f5-hello-world 8080:8080
```
Now I can see that my minikube application is available from my Windows machine, at the IP address of the Ubuntu Linux Docker host. 
![Success](/assets/accessing-minikube-remotely/port-forwarding.PNG)

`Ctrl + C` out of this, and let's find another way we can do this.

#### socat tool
[Socat](https://www.redhat.com/sysadmin/getting-started-socat) is a tool for proxying bi-directional traffic. I don't know much about it, but the article I've linked to gives a nice overview and some basic examples. In my case, here's the command I can run.

```bash
sudo ufw allow 31193 ## make sure you run this if you haven't already. Whichever port you choose to expose must not be blocked by Linux firewall.
sudo apt install socat -y
socat TCP4-LISTEN:31193,fork TCP4:192.168.49.2:31193
```
Again, we can see that the page is reachable from another machine. Note the different port I am using here (31193, not 8080, which we exposed with kubectl port-forward)

![Success](/assets/accessing-minikube-remotely/socat-tool.PNG)

#### NGINX web proxy
This one seems obvious. Here's what you would do (in all honesty, I didn't test this but I found many people who were happy with this):
1. Deploy NGINX on the Ubuntu server, listening on a port
2. Configure the upstream within NGINX to be the minikube IP address

If you want to go down this path, [here](https://zhuzean.medium.com/deploy-minikube-on-ubuntu-for-local-development-d57602f09e4b) is an article with steps you can follow.


**Note:** There's multiple ways to install Minikube. It can run as a VM on a hypervisor, but if you already have a VM, then you can just install Docker and have Minikube use Docker. That's what we've done. I'm no minikube expert, but I'm sure the options are going to be different if this is not your scenario.
{: .notice}

### Conclusion
 We've listed 3 ways you can access applications running inside Minikube from remote machines. They all involve proxying traffic (as opposed to routing traffic). Given the dev-focused nature of Minikube, it's likely that you'll have the freedom to choose which method is best for you, if your environment requires this.

 Thanks for reading!

