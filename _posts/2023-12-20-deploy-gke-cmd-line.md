---
layout: single
title:  "GKE cluster with 'GKE Dataplane V2' deploy via cmd line"
categories: [kubernetes]
tags: [gke,kubernetes]
description: "GKE cluster with GKE Dataplane-v2 create v. This is for my own notes, so I can copy and paste these in future" #this is a custom variable meant for a short description to be displayed on home page
---
<!-- begin_excerpt -->
<!-- end_excerpt -->
### GKE Dataplane V2
[GKE Dataplane V2](https://cloud.google.com/kubernetes-engine/docs/how-to/dataplane-v2) is a new eBPF-based dataplane that replaces iptables. Previously GKE used Calico but this v2 dataplane uses Cilium.

>You must enable this when deploying a cluster. This setting is immutable; cannot enable/disable this after cluster deployment.

### Create VPC Network and GKE cluster
- In the example below, I have already authenticated with gcloud cli

{% highlight bash %}
#SET VARIABLES
REGION=us-east1
CLUSTERNAME=mycluster
NETWORK=myvpcnetwork
SUBNETWORK=myvpcsubnet
RANGE='10.0.0.0/24'

#CREATE VPC NETWORK
gcloud compute networks create $NETWORK --subnet-mode=custom
gcloud compute networks subnets create $SUBNETWORK --region $REGION --range $RANGE --network $NETWORK

#CREATE CLUSTER
gcloud container clusters create $CLUSTERNAME --region $REGION --network $NETWORK --subnetwork $SUBNETWORK --enable-dataplane-v2
{% endhighlight %}

### Confirm that GKE Dataplane V2 is enabled
{% highlight bash %}
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
{% endhighlight %}

### Delete GKE cluster and VPC Network
{% highlight bash %}
#DELETE CLUSTER
gcloud container clusters delete $CLUSTERNAME --region $REGION
gcloud compute networks subnets delete $SUBNETWORK
gcloud compute networks delete $NETWORK
{% endhighlight %}

