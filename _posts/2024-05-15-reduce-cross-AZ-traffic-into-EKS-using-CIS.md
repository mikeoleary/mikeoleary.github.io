---
layout: single
title:  "Use topology labels to reduce cross-AZ ingress traffic with F5 CIS and EKS"
categories: [aws]
tags: [big-ip, aws, eks]
excerpt: "This solution outlines how to use CIS when you are looking to reduce cross-AZ traffic in AWS EKS" #this is a custom variable meant for a short description to be displayed on home page
toc: true
gallery:
  - image_path: /assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-NodePort.png
    url: /assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-NodePort.png
    title: "NodePort with typical deployment"
  - image_path: /assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-NodePort-node-label-selector.png
    url: /assets/reduce-cross-AZ-traffic-EKS/reduce-cross-AZ-traffic-NodePort-node-label-selector.png
    title: "NodePort using node-label-selector"
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

#####  Cross-AZ traffic with a typical CIS deployment
The above diagram is shows a valid deployment, but cross-AZ traffic is very likely. The BIG-IP's are unaware of K8s topology, and will load-balance equally across AZ's.
- Since only 1x BIG-IP is active in the pair, ingressing to pods in 2 out of 3 AZ's requires cross-AZ traffic
- Ingressing to pods in AZ 3 will _always_ generate cross-AZ traffic, regardless of which BIG-IP is active

#### Multiple active BIG-IP's and the node-label-selector argument with CIS
CIS can use the `node-label-selector` argument to limit load-balancing to select nodes. We will use this to keep ingress traffic local to an AZ.

To deploy an architecture like the following diagram:
- Find or create your topology labels. In my example using EKS, I see my nodes have labels such as `topology.kubernetes.io/zone=us-east-1a`
  - You can also create your own labels on nodes for this purpose
- Deploy 3x standalone BIG-IP's
- Deploy 3x CIS instances
  - Use the [```node-label-selector```](https://clouddocs.f5.com/containers/latest/userguide/config-parameters.html) argument so that each CIS instance only watches for pods on select nodes

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
          image: "f5networks/k8s-bigip-ctlr:2.16.1"
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

##### Other pointers in this solution
- NLB's are not required. To avoid NLB throughput costs, use DNS Load Balancing (GSLB) to spread traffic evenly across your Internet-facing BIG-IP's. Other methods to disaggregate traffic exist also but are not the focus of this solution.
- This solution does not require the K8s service to include any annotations, such as `service.kubernetes.io/topology-mode`. This solution assumes that the K8s pods are randomly spread across AZ's within a single service.

#### NodePort vs Cluster mode
It is worth noting that the diagrams above have assumed the CIS deployment is in ClusterIP mode, and not NodePort mode. If you were sending traffic to K8s nodes and relying on `kube-proxy` to distribute traffic evenly across pods, you would almost certainly generate cross-AZ traffc between nodes. 

{% include gallery id="gallery" caption="When using NodePort mode, you'll still generate cross-AZ traffic!"  %}

#### Further reading about topology and routing in K8s 
[Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/) is a K8s concept that uses labels and annotations to allow an administrator to define preferences for keeping traffic within regions or zones where possible. This solution does not take full advantage of this concept, but merely uses these labels to limit which pods CIS will configure on BIG-IP. Today, CIS and BIG-IP are not fully aware of the Kubernetes topology in which they operate.

It is worth noting the [AWS EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/cost_optimization/cost_opt_networking/#load-balancer-to-pod-communication), specifically as it discusses AWS Load balancers. When you use AWS ALB or NLB, you can choose between _instance mode_ (somewhat similar to the NodePort example above) and _ip mode_ (similar to the ClusterIP example above). If you're planning to do something similar to what I've done above using AWS native load balancers, make sure you understand the difference and configure accordingly.

Finally, the above solution does not apply only to AWS EKS. The concept of regions and zones exists in most clouds, and the concept of topologies in Kubernetes were created to account for these. However, it's the cross-AZ traffic charges in AWS that make this solution pressing for users of AWS EKS.

#### Summary
Consider if your external load balancer is causing cross-AZ traffic charges for ingress traffic. If using F5 CIS to populate pods as pool members in BIG-IP, consider using the `node-label-selector` argument and multiple active BIG-IP's to keep ingress traffic to pods within a single Availability Zone.

#### Footnotes
[^1]: The reasons in this case aren't important, but it's not uncommon for this kind of thing to be based on skillsets, other projects, preferred vendors, or the internal political landscape.
