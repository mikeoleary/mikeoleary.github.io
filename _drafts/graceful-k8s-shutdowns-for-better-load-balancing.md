---
layout: single
title:  "Making K8s shutdowns more graceful"
categories: [kubernetes]
tags: [azure,linux]
excerpt: "This post discusses measures a K8s admin can take to allow for more graceful shutdowns, using preStop hooks and terminationGracePeriodSeconds." #this is a custom variable meant for a short description to be displayed on home page
---
When requests are sent to pods in Kubernetes by proxies such as external load balancers or ingress controllers, these proxies must be configured with the IP address of each pod in the exposed service[^1]. Likewise, they must be updated if the pod IP addresses change, such as when deployments are scaled in/out or when Kubernetes replaces a failed pod with a new one. 

When a pod is deployed or terminated, this information must be quickly used to configure any proxy serving network traffic to this pod. However, we cannot update the proxy before we know certain information. For example, if a new pod is deployed, we at least must know it's IP address before we update any proxy with this information. If a pod has failed, the Kubernetes management plane can terminate the pod and deploy a new pod, but any proxy that learns this information from the Kubernetes API will be receiving this information at the same time - or more likely after the time - that the Kubernetes management plane takes action. 

As you can see, there is a small window of time when the proxy's targets (i.e, the IP addresses of pods in a service) are not synchronized with the true state of running pods in a service. Usually this window is extremely short - perhaps a second or two is typical for ingress controllers - but it can also be longer for external load balancers. **In either case, if a request arrives at an ingress controller or external load balancer during this window, it may be routed to a pod that has failed, is being terminated, or no longer exists, resulting in a failed request.**

This article describes measures a K8s admin can take to improve upon this situation. While we can never fully guarantee that a request in transit will not arrive at a pod that is in a failed or terminating state (or a pod that no longer exists!), we can architect for a smoother orchestration of the events involved in updating external load balancers or ingress controllers.

### Out of scope: K8s health probes
This article is not concerned with [liveness, readiness, or startup probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#types-of-probe). Knowledge of these probes and correct use of them is important, and will go a long way toward your proxy being configured correctly. However, even with your probes configured and working correctly, our timing issue can still occur: there is still a short window of time when requests may be routed to a failed pod because a proxy has not yet been updated.

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

### Endpoints and Services
The Kubernetes [```Endpoints```](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints) object is a separate object from a [```Service```](https://kubernetes.io/docs/concepts/services-networking/service/#services-in-kubernetes), although there is an ```Endpoints``` object for every ```Service``` object. While a ```Service``` is a group of pods that should be reachable via their IP addresses and a port, the ```Endpoints``` of that service is a list of these pods that have satisfied their Readiness probe. This means you could have a long list of pods in a ```Service```, but a shorter list of IP addresses in an ```Endpoints``` object, if some pods in that service have failed.



### Related articles
- https://learnk8s.io/graceful-shutdown
- https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace

[^1]: This is true for services of type ClusterIP and LoadBalancer, where the load balancer is aware of individual pod IP addresses. This is not true for services of type NodePort, where the load balancer will direct traffic only to node IP addresses.

