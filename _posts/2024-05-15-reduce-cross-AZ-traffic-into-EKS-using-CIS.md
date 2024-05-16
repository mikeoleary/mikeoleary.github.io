---
layout: single
title:  "Reducing cross-AZ traffic when using F5 CIS and EKS"
categories: [aws]
tags: [big-ip, aws, eks]
excerpt: "This solution outlines how to use CIS when you are looking to reduce cross-AZ traffic in AWS EKS" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
#### Requirements and background
Recently I had a customer with the following environment:
- the need to load-balance traffic from Internet-based clients into EKS
- the need for mTLS termination and other functionality that required BIG-IP
- AWS EKS cluster deployed in 3x Availability Zones (AZ's)

Simple enough, right? But there's some other problems that add an interesting twist to the requirements:

- **reduce or eliminate cross-AZ traffic** between any external load balancer and EKS (due to cost of cross-AZ traffic)
- **do not use NLB** if possible (NLB throughput cost is significant)
- while mTLS/other functionality could be performed _inside_ the cluster (eg NGINX Ingress Controller), we want these functions performed external to the cluster (for "business" reasons)[^1]

Let's solve for this!

#### F5 CIS with a typical, default installation
##### Typical CIS deployment
Typically, a customer will use F5 CIS to dynamically update the configuration of an HA pair of BIG-IP's, sending traffic directly to pods running inside Kubernetes (K8s).

<figure>
    <a href="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-default.png" class="image-popup" title="Typical HA pair deployment.">
        <img src="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-default.png">
    </a>
    <figcaption>This typical HA-pair deployment would be easy, but it would not meet all requirements.</figcaption>
</figure>

#####  Problems with a typical CIS deployment
There main problem with the design above is that **cross-AZ traffic is very likely**:
- The BIG-IP's are unaware of pod topology and will load-balance equally across AZ's
- Pods in AZ 3 can **only** be reached by generating cross-AZ traffic
- Since only 1x BIG-IP is active, cross-AZ traffic would occur for about 2 out of every 3 connections!

<figure>
    <a href="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-default.png" class="image-popup" title="3-AZ deployment with NodePort.">
        <img src="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-default.png">
    </a>
    <figcaption>A typical BIG-IP HA pair with CIS configuring the BIG-IP devices.</figcaption>
</figure>

#### NodePort vs Cluster mode
It is worth noting that the diagrams above have assumed your CIS deployment is in ClusterIP mode, and not NodePort mode. If you were sending traffic to K8s nodes and relying on ```kube-proxy``` to distribute traffic evenly across pods, you would almost certainly generate a high amount of cross-AZ traffc between nodes. 

<figure>
    <a href="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-NodePort.png" class="image-popup" title="3-AZ deployment with NodePort.">
        <img src="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-NodePort.png">
    </a>
    <figcaption>Don't do this with NodePort mode! You'll still generate cross-AZ traffic!</figcaption>
</figure>

#### Using the node-label-selector argument with CIS
Today, CIS does not take advantage of native [Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/), a K8s feature that is still in beta. However, the ```node-label-selector``` argument achieves a very similar outcome: keeping traffic local to an AZ where possible.

To deploy an architecture like the following diagram:
- Labels your nodes as appropriate. 
  - You can do this automatically using node groups. 
  - You likely already have node labels you can use. Eg., using EKS, I see my nodes have labels such as `topology.kubernetes.io/zone=us-east-1a` which I will use
- Deploy 3x standalone BIG-IP's
- Deploy 3x CIS instances, one for each BIG-IP. Use the ```node-label-selector``` argument so that each CIS instance only watches for pods on select nodes (CIS arguments are documented [here](https://clouddocs.f5.com/containers/latest/userguide/config-parameters.html)).

<figure>
    <a href="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-node-label-selector.png" class="image-popup" title="3-AZ deployment with ClusterIP.">
        <img src="/assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-node-label-selector.png">
    </a>
    <figcaption>Notice that this design uses 3x standalone BIG-IPs and limits ingress traffic within an AZ.</figcaption>
</figure>

Here is an example of a CIS deployment with a `node-label-selector` (line 32):
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: f5cis1
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-bigip-ctlr-deployment
  template:
    metadata:
      labels:
        app: k8s-bigip-ctlr-deployment
    spec:
      containers:
        - name: k8s-bigip-ctlr
          image: "f5networks/k8s-bigip-ctlr:2.16.0"
          env:
            - name: BIGIP_USERNAME
              valueFrom:
                secretKeyRef:
                  name: bigip-login
                  key: username
            - name: BIGIP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: bigip-login
                  key: password
          command: ["/app/bin/k8s-bigip-ctlr"]
          args: [
            "--node-label-selector=topology.kubernetes.io/zone=us-east-1a",
            "--bigip-username=$(BIGIP_USERNAME)",
            "--bigip-password=$(BIGIP_PASSWORD)",
            "--bigip-url=10.0.0.11",
            "--bigip-partition=kubernetes",
            "--pool-member-type=cluster",
            "--insecure",
            "--custom-resource-mode=true",
            "--log-level=DEBUG",
            "--disable-teems=true"
            ]
      serviceAccount: bigip-ctlr
      serviceAccountName: bigip-ctlr
      imagePullSecrets:
        - name: bigip-login

```



#### Further reading about topology and routing in K8s 
K8s allows for [topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) to define how you would like your pods deployed across zones. Similar to pod and node affinity and anti-affinity, this allows administrators to consider which nodes their pods will be scheduled to. This article will not dive deep into Kubernetes topology, except to say that it is set up using node labels called `topologyKey` labels.

It is worth noting the [AWS EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/cost_optimization/cost_opt_networking/#load-balancer-to-pod-communication), specifically discussing AWS Load balancers. When you use AWS ALB or NLB, you can choose between _instance mode_ (somewhat similar to the NodePort example above) and _ip mode_ (similar to the ClusterIP example above). If you're planning to do something similar to what I've done above, but using AWS native load balancers, make sure you understand the difference and configure accordingly. 

Finally, the above solution does not apply only to AWS EKS. The concepts of zones exists in most clouds, and the concept of topologies in Kubernetes were created to account for these. However, it's the cross-AZ traffic charges from AWS that make this use case pressing for AWS EKS users.

#### Summary
When running your K8s workloads across multiple AZ's, consider if your external load balancer is causing cross-AZ traffic. If using F5 CIS to populate pool members in BIG-IP, consider using the `node-label-selector` argument to ensure that BIG-IP sends traffic only to pods within the same Availability Zone.

#### Footnotes
[^1]: The reasons in this case aren't important, but it's not uncommon for this kind of thing to be based on skillsets, other projects, preferred vendors, or the internal political landscape.
