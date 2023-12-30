---
layout: single
title:  "Making K8s shutdowns more graceful"
categories: [kubernetes]
tags: [kubernetes]
excerpt: "This post discusses measures a K8s admin can take to allow for more graceful shutdowns, using preStop hooks and terminationGracePeriodSeconds." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
### Summary
When pods are removed from Kubernetes services, there is a short time window when requests may still be routed to them by ingress controllers, external load balancers, or other components. This results in failed requests. As Kubernetes admins, we can make small changes to make shutdowns and rolling upgrades more graceful for end users.

### Covering the basics

#### Traffic eventually reaches a pod
Pods have one or more containers that run applications such as web servers and databases. Pods have IP addresses and in many cases must be reachable by clients outside of the cluster to serve requests. To route traffic to pods we typically point client traffic toward an ingress controller first, which then routes the traffic to the appropriate pod based on Kubernetes objects like services and endpoints.

#### Services and Endpoints
A ```Service``` is a group of pods that should be reachable by clients. The ```Endpoints``` of that service is a list of the service's pods' IP addresses and ports, but only those that have satisfied their readiness probe. The Kubernetes [Endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints) object is separate from a [Service](https://kubernetes.io/docs/concepts/services-networking/service/#services-in-kubernetes) object, although for every ```Service``` there is an ```Endpoints```.

#### Readiness probes
It's important to know about [liveness, readiness, and startup probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#types-of-probe). Startup and liveness probes may determine if a pod is considered `Running`, but it's the **readiness probe** that determines if a pod is *ready to receive traffic*. If a pod passes it's readiness probe (or doesn't have a readiness probe configured), it's IP address and port are added to the `Endpoints` object for the corresponding service.

### How requests reach pods in Kubernetes
When requests are sent to pods in Kubernetes via proxies such as external load balancers or ingress controllers, these proxies must be configured with the IP address of each pod in the exposed service[^1]. Likewise, they must be updated if the pod IP addresses change, such as when deployments are scaled in/out or when Kubernetes replaces a failed pod with a new one. And again, when a pod's expected lifecycle is complete - whether the reason is a completed task or a rolling update - the pod's IP address must be removed from the proxy.

I've used the term *proxy* to generally refer to ingress controllers, external load balancers, and operators that subscribe to the Kubernetes API server and watch for changes to Kubernetes services[^2]. In this post I'll use three examples of proxies that I know pretty well:
- NGINX Ingress Controller: *an ingress controller*
- F5 Distributed Cloud (XC) Customer Edge(CE): *can act as external load balancer* [^3]
- F5 Container Ingress Services (CIS): *an operator that updates a BIG-IP device outside the cluster*

#### Ingress Controller
Ingress controllers typically run as a set of pods that use a Service Account (SA). This SA has a ClusterRole that allows it to get, list, and [watch](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes) Services and Endpoints. Because an ingress controller is a pod itself and is watching the Kubernetes API closely and reconfiguring itself immediately, the time lapse between an endpoint being removed and the proxy configuration being updated is very short.

#### External load balancer
A F5 XC CE device can use a Service Account to authenticate to the Kubernetes API server from outside of the cluster. I've [written](https://community.f5.com/t5/technical-articles/using-a-kubernetes-serviceaccount-for-service-discovery-with-f5/ta-p/300225) about this before. In this setup, the XC device can configure itself to load balance incoming requests directly to pod IP addresses within a cluster. Users can send requests to this device (or actually any device on the mesh from which the site is advertised) and their request will be proxied directly to a pod's IP address.

Depending on the frequency of API calls into the cluster, and whether they are designed to get or subscribe to events (the `watch` verb maintains an open connection and streams updates to clients), there may be a longer time lapse between a pod's termination and the load balancer learning about this.

#### Operator
F5 CIS is an enterprise-level operator and a good example of real-world complexities. CIS runs as a container within a cluster and subscribes to the Kubernetes API, just like other clients. Unlike an ingress controller that configures itself, upon learning of changes to endpoints CIS will send an API call to an F5 BIG-IP device that is *outside* of the cluster. 

This operation itself may take a second or so, but changes are sometimes deliberately batched and sent at a configured frequency, such as every 30 seconds (in cases where there are frequent changes, for example). If this were the case, an external BIG-IP may not know about a failed or terminated pod for up to 30 seconds, plus whatever processing time the API calls require.

### How this time lapse occurs on pod deletion but not creation

When a pod is deployed, it's IP and port must be quickly used to configure any proxy serving network traffic to this pod. However, we learn this information by querying Endpoints, not Services or Pods themselves. When a pod is deployed, it must pass it's startup, liveness, and readiness checks before it is added to an Endpoints object. In this way, we know that any pod IP address that is added to a proxy must already be healthy and ready to serve requests.

However, if a pod has failed, the Kubernetes management plane will remove it from the Endpoints object. 

As you can see, there is a small window of time when the proxy's targets (i.e, the IP addresses of pods in a service) are not synchronized with the true state of running pods in a service. Usually this window is extremely short - perhaps a second or two is typical for ingress controllers - but it can also be longer for external load balancers. **In either case, if a request arrives at an ingress controller or external load balancer during this window, it may be routed to a pod that has failed, is being terminated, or no longer exists, resulting in a failed request.**

This article describes measures a K8s admin can take to improve upon this situation. While we can never fully guarantee that a request in transit will not arrive at a pod that is in a failed or terminating state (or a pod that no longer exists!), we can architect for a smoother orchestration of the events involved in updating external load balancers or ingress controllers.



### These aren't the droids you're looking for: pod lifecycle phases
Kuberentes pods have well-defined lifecycle [phases](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase). They are below.

|Value|Description|
|---|---|
|Pending|	The Pod has been accepted by the Kubernetes cluster, but one or more of the containers has not been set up and made ready to run. This includes time a Pod spends waiting to be scheduled as well as the time spent downloading container images over the network.|
|Running|	The Pod has been bound to a node, and all of the containers have been created. At least one container is still running, or is in the process of starting or restarting.|
|Succeeded|	All containers in the Pod have terminated in success, and will not be restarted.|
|Failed|	All containers in the Pod have terminated, and at least one container has terminated in failure. That is, the container either exited with non-zero status or was terminated by the system.|
|Unknown|	For some reason the state of the Pod could not be obtained. This phase typically occurs due to an error in communicating with the node where the Pod should be running.|

However, see the following note from Kubernetes documentation:

>Note: When a Pod is being deleted, it is shown as ```Terminating``` by some kubectl commands. This ```Terminating``` status is not one of the Pod phases. A Pod is granted a term to terminate gracefully, which defaults to 30 seconds. You can use the flag ```--force````` to [terminate a Pod by force](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)

This article primarily focuses on situations where a Pod fails or is terminated by the kubelet. Since ```Terminating``` is not a Pod phase, proxies cannot learn pod readiness from pod lifecycle phases alone.





### Related articles
- https://learnk8s.io/graceful-shutdown
- https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace

[^1]: This is true for services of type ClusterIP and LoadBalancer, where the load balancer is aware of individual pod IP addresses. This is not true for services of type NodePort, where the load balancer will direct traffic only to node IP addresses.
[^2]: There are also other components within Kubernetes that do not proxy traffic, but must subscribe to changes to endpoints. For example, CoreDNS must keep track of IP addresses for [headless services](https://kodekloud.com/blog/kubernetes-headless-service/), and the [Cloud Controller Manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/) component of the control plane will need to know pod IP addresses in order to create cloud load balancers. 
[^3]: The F5 Distributed Cloud Customer Edge device can act as an external LB for existing Kubernetes clusters, but it can also be deployed *within* Kubernetes as an applicatin itself, or even run it's own Kubernetes cluster. In this case, I'm using it as an example of a third party external load balancer to Kubernetes.

