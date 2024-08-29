---
layout: single
title:  "FTP server on K8s with F5"
categories: [kubernetes]
tags: [kubernetes, openshift]
excerpt: "FTP is a legacy protocol that requires specific knowledge. Most young engineers have never used it. Kubernetes is modern, but complicated. Most senior engineers have never used it. Let's do this!" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

### Background
After 5 years honing Kubernetes expertise, I was happy to undertake a challenge: expose an FTP server from within Kubernetes, using Azure Red Hat OpenShift (ARO), and protecting with F5 BIG-IP. Here's how I did this.

### Isn't FTP a legacy technology?
Yes, FTP has been around since the early 70's. It was designed for efficient tranfer of files, and although it's insecure by default, it's still commonly seen today.

#### Advantages of FTP
As opposed to other file transfer methods, FTP does offer a few advantages:
- allows applications to resume file transfers if a connection is lost
- allows for a queue of files to be uploadeded or downloaded
- faster / more efficient than HTTP
- no file size limitations
- extremely common platform offered widely for many years

#### Disadvantages of FTP
FTP is considered legacy because of a few limitations. There's workaround and enhancements, so I'll give a basic overview:
- insecure by default
  - Standard FTP sends username, password, and files in clear text
  - Servers can be spoofed to send data to the wrong client
- complexity
  - by default, the control channel is over TCP/21, but is usually configurable
  - by default, the data channel is over a random high port, if using Passive mode, and is usually configurable
  - by default, the data channel is over TCP/20, if using Active mode, and is usually configurable
- active vs passive
  - active mode requires the server to establish a connection to a client. This is an outdated model disallowed by most firewalls
  - passive mode requires outbound connections over random high ports, which are also disallowed by most firewalls. Therefore, firewalls or L4 network devices (like BIG-IP) must be "ftp aware"

There are many more complex advantages and disadvantages, but to summarize: running and securing FTP servers requires knowledge of the protocols, not just general network and security knowledge.

#### Enhancements and common FTP servers
FTP is common, but it's also common to see enterprises also offer FTPS and SFTP. While they sound similar, these are different protocols. FTPS allows for encryption of the control channel and (optionally) the data channel. SFTP adds file transfers upon the SSH protocol, meaning all transfers can happen over a single TCP connection on port 22. However, SFTP is difficult to proxy, and FTPS requires additional knowledge on top of FTP.

For this reason, enterprise-level FTP servers are usually well built out with a customer support team, static IP addresses, commercial software, and support. Changes are usually slow and upgrades infrequent. File transfers are usually frequent, large, and core to a business process. For example, large financial customers may have longstanding practices built around FTP, and so require very high confidence in the redundancy, security, and support of their FTP systems.

In my most recent case, my customer was running IBM Sterling B2B Integrator installed on Azure Red Hat OpenShift (ARO). This is an enterprise-level, commercial application running an enterprise K8s distribution. In my demonstration, I'm going to use a free FTP server, vsftp, but I will run this on ARO also, to be as close to an enterprise use case as possible.

### Why run FTP on Kubernetes?

Kubernetes offers the same advantages for FTP applications as it does to others: scale, platform-agnostic, heavily automated deployment and operations, etc. But Kubernetes networking is a challenge for the average network engineer, and FTP applications are an *additional* challenge for the average person. FTP and K8s are not a likely combinatino, but it can be done!

### How to run your FTP server on Kubernetes and still get enterprise-level protection

I've [written](https://community.f5.com/kb/technicalarticles/running-f5-with-managed-azure-redhat-openshift/291157) about how to run F5 on ARO in the past. Let's set up an ARO environment following [this tutorial](https://learn.microsoft.com/en-us/azure/openshift/create-cluster?tabs=azure-cli) using az cli.

````bash
LOCATION=eastus2
RESOURCEGROUP=oleary-follett-rg
CLUSTER=myarocluster
# create vnet
az network vnet create --resource-group $RESOURCEGROUP --name aro-vnet --address-prefixes 10.0.0.0/16 --location $LOCATION
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name master-subnet --address-prefixes 10.0.0.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name worker-subnet --address-prefixes 10.0.2.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name mgmt --address-prefixes 10.0.4.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name external --address-prefixes 10.0.6.0/23
# create ARO environment. This will take around 45 mins to complete
az aro create --resource-group $RESOURCEGROUP --name $CLUSTER --vnet aro-vnet --master-subnet master-subnet --worker-subnet worker-subnet
# when completed, get the kubeconfig file from the ARO environment and replace your kubeconfig file with this
az aro get-admin-kubeconfig -g $RESOURCEGROUP -n $CLUSTER --file ~/.kube/config
````
<hr style="border-color: gray">
Now, deploy a pair of F5 BIG-IP devices into the VNET, where the network interfaces are in the subnets of mgmt, external, and worker-subnet.[^1]

<hr style="border-color: gray">
Now, configure CIS in the cluster so that applications can be exposed from Kubernetes via BIG-IP.[^2]

<hr style="border-color: gray">
Now, deploy your FTP server like this:

1. First, a namespace and service account to use
2. Second, a PersistentVolumeClaim like this.
3. Then, a Deployment like this to run a FTP server.
  a. Reference the container image of docker.io/fauria/vsftpd
  b. In the environment variables
    1. use the Public IP address that matches the frontendipconfig on the Azure LB.
    2. set a min and max for high port ranges
4. Now, create a service of type NodePort. Like this.
5. Finally, in order to have CIS create a VirtualServer on BIG-IP, create a ConfigMap like this.

Now, you have an architecture that looks like this:

### Testing our FTP application

I'm going to use a simple ftp commands at the linux command prompt. Below, I will connect to our FTP server with <code>ftp -p 4.152.28.199</code>, and then enter username and password when prompted. Then I will type <code>ls</code> in order to list the directory, which contains a single file `test.txt`. Finally, I will type <code>bye</code> to disconnect.

````bash
ubuntu@ubuntu-Virtual-Machine:~$ ftp -p 4.152.28.199
Connected to 4.152.28.199.
220 (vsFTPd 3.0.2)
Name (4.152.28.199:ubuntu): vsftp
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||32002|)
150 Here comes the directory listing.
-rw-------    1 ftp      ftp             8 Aug 29 01:51 test.txt
226 Directory send OK.
ftp> bye
221 Goodbye.
````

Here's a very short clip using a graphical tool, WinSCP, to demonstrate the same thing:

<video width="480" height="320" controls="controls">
  <source src="/assets/ftp-on-k8s/vsftp-in-k8s-with-sound.mp4" type="video/mp4">
</video>

### Conclusion

It is possible to:
* run FTP applications in Kubernetes
* expose FTP services outside of the cluster
* integrate external services, like F5 BIG-IP, with the FTP traffic
* run this in public cloud or with enterprise services like Azure Red Hat OpenShift

Most of this requires a skillset that covers legacy and modern technologies. If you undertake something like this, ensure you have a plan for high availability and commercial support. Thanks for reading!

<hr style="border-color: gray">
{:footnotes}
* 
[^1]: For this step, I typically deploy an ARM template from F5, like this one: https://github.com/F5Networks/f5-azure-arm-templates-v2/tree/main/examples/failover
[^2]: This is more complex than just installing an operator, and requires BIG-IP credentials. I won't go into the details here.

