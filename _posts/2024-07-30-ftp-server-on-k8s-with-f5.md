---
layout: single
title:  "FTP server on K8s with F5"
categories: [kubernetes]
tags: [kubernetes]
excerpt: "FTP is a legacy protocol that requires specific knowledge. Most young engineers have never used it. Kubernetes is modern, but complicated. Most senior engineers have never used it. Let's do this!" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

### Background
After 5 years honing Kubernetes expertise, I was happy to undertake a challenge: expose an FTP server from within Kubernetes, and protecting with F5 BIG-IP. I'll do this using Azure Kubernetes Service (AKS) as an example environment.

**Isn't FTP a legacy technology?** Yes, FTP has been around since the early 70's. It was designed for efficient tranfer of files. Although it's insecure by default, it's still commonly seen today for some file transfer types.
{: .notice--info}

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

#### FTPS and SFTP
FTP is common, but it's also common to see enterprises also offer FTPS and SFTP. While they sound similar, these two are different protocols. FTPS allows for encryption of the control channel and (optionally) the data channel of FTP. SFTP adds file transfers upon the SSH protocol, meaning all transfers can happen over a single TCP connection on port 22. 

There is some difficulty with all of these technologies. SFTP is difficult to proxy. FTPS requires additional knowledge on top of FTP, which itself is a learning curve for most folks.

#### Common FTP servers

For this reason, enterprise-level FTP servers are usually well built out with a customer support team, static IP addresses, commercial software, and support. Changes are usually slow and upgrades infrequent. File transfers themselves are usually frequent, large, and core to a business process. For example, large financial customers may have longstanding practices built around FTP with partners, and so require very high confidence in the redundancy, security, and support of their FTP systems.

In my most recent case, my customer was running [IBM Sterling B2B Integrator installed Red Hat OpenShift](https://community.ibm.com/community/user/dataexchange/blogs/nikesh-midha1/2020/02/19/ibm-sterling-b2b-integrator-on-red-hat-openshift-c). OpenShift was running on Azure ([ARO](https://learn.microsoft.com/en-us/azure/openshift/intro-openshift)). This means we had a commercial application running on an enterprise K8s distribution, which itself was running as a managed service in Azure. In my demonstration, I'm going to use a free FTP server, vsftp, and I will start by running on Azure Kubernetes Service (AKS). 

### Why run FTP on Kubernetes?

Kubernetes offers the same advantages for FTP applications as it does to others: scale, platform-agnostic, heavily automated deployment and operations, etc. But Kubernetes networking is a challenge for the average network engineer, and FTP applications are an *additional* challenge for the average engineer. FTP and K8s are not a likely combination, but it can be done!

### How to run your FTP server on Kubernetes and still get enterprise-level protection

#### Service type of ClusterIP, NodePort, or LoadBalancer?

Broadly speaking, there's three major types of services in Kubernetes[^1]: ClusterIP, NodePort, and LoadBalancer. Whichever type you choose to expose your FTP service, you're going to have some things to consider.

<figure>
    <a href="/assets/ftp-on-k8s/3-types-services.webp"><img src="/assets/ftp-on-k8s/3-types-services.webp"></a>
    <figcaption>3 types of LB services. Image <a href="https://opensource.com/article/22/6/kubernetes-networking-fundamentals">source</a>.</figcaption>
</figure>

##### ClusterIP
[Cluster IP](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) is possible if your pods are directly routable by your external load balancer. With my AKS deployments, my pods are routable from the VNet without additional work.

If you're using a CNI with an overlay network (which typically use tunnels like VXLAN or GENEVE or routing like BGP), then you'll need to get your external loadbalancer integrated with the CNI, or use another method. I won't go into the details of CNI integration here.

##### NodePort
[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) builds on top of ClusterIP, and maps a high port from the NodePort range (30000-32767) to port exposed on the pod. 

However, with FTP, we _**cannot**_ have random port translation. Since the data channel port is assigned by the server, it will tell the client to connect on an expected port. That port must be available to the client, and correctly mapped back to the server.

The way to do this is to [manually define](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport-custom-port) our NodePort values in our service, and match them with the PASV ports from your FTP server. Ie., 

````yaml
apiVersion: v1
kind: Service
metadata:
  name: my-ftp-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:    
    - port: 21
      targetPort: 21
      # no need for a pre-determined port for the control channel
    - port: 30100
      targetPort: 30100
      nodePort: 30100 # notice here, we have our PASV port range within the NodePort default range, and we match them manually.
    - port: 30101
      targetPort: 30101
      nodePort: 30101
    # etc...
````

In reality, a NodePort service is a confusing option for exposing FTP services!

##### LoadBalancer
Creation of a [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) service builds upon NodePort. A [controller](https://kubernetes.io/docs/concepts/architecture/controller/) will automatically configure a corresponding load balancer in the cloud. 

In this case, I'm running in Azure but I *do not* want to use an Azure Load Balancer, so I'll create a service of type LoadBalancer but also define [`loadBalancerClass`](https://clouddocs.f5.com/containers/latest/reference/release-notes.html#release-notes-2-18-0) so the Azure controller ignores this object. CIS will configure BIG-IP as the load balancer instead. 

<figure>
    <a href="/assets/ftp-on-k8s/services.png"><img src="/assets/ftp-on-k8s/services.png"></a>
    <figcaption>LoadBalancer is a superset of NodePort, which is itself a superset of ClusterIP.</figcaption>
</figure>

#### Let's deploy our cloud environment

In this demo, I'll use a LoadBalancer service and deploy my CIS instance in cluster mode.

Build an Azure VNet with a few subnets. 

````bash
SUBSCRIPTION_ID="your-subscription-id"
LOCATION=eastus2
RESOURCEGROUP=oleary-rg
CLUSTER=mycluster
VNET_NAME=my-vnet
# create vnet
az network vnet create --resource-group $RESOURCEGROUP --name $VNET_NAME --address-prefixes 10.0.0.0/16 --location $LOCATION
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $VNET_NAME --name worker-subnet --address-prefixes 10.0.2.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $VNET_NAME --name mgmt --address-prefixes 10.0.4.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $VNET_NAME --name external --address-prefixes 10.0.6.0/23
````
<hr style="border-color: gray">

Now, deploy a pair of F5 BIG-IP devices into the VNET, where the network interfaces are in the subnets of `mgmt`, `external`, and `worker-subnet`.[^2]

<hr style="border-color: gray">

Then, deploy an AKS cluster with the nodes in the `worker-subnet` subnet.
````bash
# create AKS cluster
az aks create --resource-group $RESOURCEGROUP --name $CLUSTER --node-count 1 --generate-ssh-keys --network-plugin azure --service-cidr "172.16.0.0/24" --dns-service-ip "172.16.0.10" --vnet-subnet-id /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/worker-subnet
# get kubeconfig file of AKS cluster
az aks get-credentials -n $CLUSTER -g $RESOURCEGROUP -f ~/.kube/config
````

<hr style="border-color: gray">

Now, configure CIS in the cluster so that applications can be exposed from Kubernetes via BIG-IP.[^3]

I'm going to use Cluster mode (not NodePort mode) in this example, but either will work.[^4]

<hr style="border-color: gray">

At this point, you'll have an environment that looks like this:
<figure>
    <a href="/assets/ftp-on-k8s/ftp-in-k8s-1.png"><img src="/assets/ftp-on-k8s/ftp-in-k8s-1.png"></a>
    <figcaption>Typical BIG-IP integration with K8s using CIS</figcaption>
</figure>

<hr style="border-color: gray">
Now, deploy your FTP server like this:

1. First, a namespace like this.
2. Second, a PersistentVolumeClaim like this.
3. Then, a Deployment like this to run a FTP server. Notice that our deployment defines several environment variables within our pods, which are used to set the PASV FTP ports and the FTP server address.
4. On the BIG-IP, create an iRule called `/Common/ftp_ports` that looks like this: `when SERVER_CONNECTED { FTP::port 10000 10002 }`
5. Now, create a policy object like this.
6. Finally, in order to have CIS create a VirtualServer on BIG-IP, a service of type LoadBalancer like this.

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

### OpenShift vs other Kubernetes distributions

You may notice in my example above that I have used `fauria/vsftp` as the container image for my FTP server. This will work in a regular K8s distro (I've used AKS in my PoC). OpenShift will require additional resources, such as a Security Context Constraint (SCC), so I have not documented this here. Perhaps in a future article.

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
[^1]: There are more than 3 types of services in K8s, but understanding these 3 major types is key.
[^2]: For this step, I typically deploy an ARM template from F5, like this one: https://github.com/F5Networks/f5-azure-arm-templates-v2/tree/main/examples/failover
[^3]: I won't detail installing CIS here, except to say that I defined `pool-member-type` as `cluster` and `load-balancer-class` as `f5cis`, to match the spec in my service.
[^4]: I find cluster mode easiest. If using a service of type NodePort, there are several differences. The PASV ports must be between 30000-32767, must manually match each NodePort they are assigned, and the FTP server must send the IP address of the K8s Node (not Pod) in the PASV response. CIS must have `pool-member-type` as `nodeport`



