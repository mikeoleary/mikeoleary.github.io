---
layout: single
title:  "XC K8s sites deployed quickly"
categories: [kubernetes]
tags: [gke, xc]
excerpt: "These are shortcut commands for when I want to deploy a XC site in K8s quickly." #this is a custom variable meant for a short description to be displayed on home page
toc: true
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

### AKS

#### AKS cluster deployment
I like to create a VNET when creating a cluster:

{% highlight bash %}
#SET VARIABLES
REGION=eastus2
CLUSTER_NAME=oleary-cluster
VNET_NAME=oleary-vnet
VNET_CIDR='10.0.0.0/16'
RG_NAME=oleary-rg2
SUBNET_NAME=subnet1
SUBNET_CIDR='10.0.0.0/24'

#CREATE VNET
az group create --location $REGION -n $RG_NAME

az network vnet create \
    --name $VNET_NAME \
    --resource-group $RG_NAME \
    --address-prefix $VNET_CIDR \
    --subnet-name $SUBNET_NAME \
    --subnet-prefixes $SUBNET_CIDR \
    --location $REGION

az aks create \
    --resource-group $RG_NAME \
    --name $CLUSTER_NAME \
    --node-count 1 \
    --node-vm-size="Standard_D4s_v3"
    --vnet-subnet-id "/subscriptions/aacd7ba7-e47c-4cb7-a8b7-90f81fdd3865/resourceGroups/$RG_NAME/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$SUBNET_NAME" \
    --service-cidr "172.16.0.0/16" \
    --dns-service-ip "172.16.0.10" \
    --network-plugin azure 

#Note: cluster must have nodes with at least 4vCPU for XC site to deploy.

az aks get-credentials --resource-group $RG_NAME --name $CLUSTER_NAME

{% endhighlight %}


#### XC site deployment

{% highlight bash %}

curl --output ce-k8s.yml "https://gitlab.com/volterra.io/volterra-ce/-/blob/master/k8s/ce_k8s.yml"

#edit the manifest and change 3 things: Lat/Long, name of XC site, and site token.

kubectl apply -f ce_k8s.yml

{% endhighlight %}


<!-- 
{% highlight bash %}
#code sample here
{% endhighlight %}
-->

