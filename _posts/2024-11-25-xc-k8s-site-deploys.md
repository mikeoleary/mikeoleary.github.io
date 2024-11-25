---
layout: single
title:  "XC K8s sites deployed quickly"
categories: [kubernetes]
tags: [gke, xc]
excerpt: "These are shortcut commands for when I want to deploy a XC site in K8s quickly." #this is a custom variable meant for a short description to be displayed on home page
---
This post exists solely to allow me to copy/paste some quick commands to deploy a K8s cluster and K8s site in F5 Distributed Cloud (XC).

### GKE

#### GKE cluster deployment
I like to create a VPC when creating a cluster:

{% highlight bash %}
#SET VARIABLES
REGION=us-east1
CLUSTERNAME=oleary-cluster
NETWORK=myvpcnetwork
SUBNETWORK=myvpcsubnet
RANGE='10.0.0.0/24'

#CREATE VPC NETWORK
gcloud compute networks create $NETWORK --subnet-mode=custom
gcloud compute networks subnets create $SUBNETWORK --region $REGION --range $RANGE --network $NETWORK

gcloud container clusters create oleary-cluster \
   --machine-type=e2-standard-4 \
   --region us-east1

#Note: cluster must have nodes with at least 4vCPU for XC site to deploy.

{% endhighlight %}

#### XC site deployment

{% highlight bash %}

curl --output ce-k8s.yml "https://gitlab.com/volterra.io/volterra-ce/-/blob/master/k8s/ce_k8s.yml"

#edit the manifest and change 3 things: Lat/Long, name of XC site, and site token.

kubectl apply -f ce-k8s.yml

{% endhighlight %}


<!-- 
{% highlight bash %}
#code sample here
{% endhighlight %}
-->

