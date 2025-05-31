---
layout: single
title:  "Azure Red Hat OpenShift (ARO) deploy script"
categories: [openshift]
tags: [kubernetes, openshift, azure]
excerpt: "This is purely for personal use to save me time on this repetitive task." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/aro/aro-azure-header-image.png"><img src="/assets/aro/aro-azure-header-image.png"></a>
</figure>

### Summary
This post is purely for myself to copy a script in future to set up an OpenShift lab in a hurry.

This script will:
- assume you have a pull secret
- deploy a VNET with 5 subnets (mgmt, external, internal, master-subnet, worker-subnet)
- deploy ARO cluster into `master-subnet` and `worker-subnet`

This script will not: 
- complete the F5 BIG-IP deployment
- complete CIS setup

### Final goal of lab build
We want to build this architecture to test out various CIS features:

<figure>
    <a href="/assets/aro/ARO-with-f5.png"><img src="/assets/aro/ARO-with-f5.png"></a>
</figure>

### az cli script
```bash
LOCATION=eastus2                 # the location of your cluster
RESOURCEGROUP=oleary-aro-rg            # the name of the resource group where you want to create your cluster
CLUSTER=mycluster                 # the name of your cluster

az group create --name $RESOURCEGROUP --location $LOCATION

az network vnet create --resource-group $RESOURCEGROUP --name aro-vnet --address-prefixes 10.0.0.0/16
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name mgmt --address-prefixes 10.0.0.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name external --address-prefixes 10.0.2.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name internal --address-prefixes 10.0.4.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name master-subnet --address-prefixes 10.0.6.0/23
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name worker-subnet --address-prefixes 10.0.8.0/23

az aro create --resource-group $RESOURCEGROUP --name $CLUSTER --vnet aro-vnet --master-subnet master-subnet --worker-subnet worker-subnet --pull-secret @pull-secret.txt
```