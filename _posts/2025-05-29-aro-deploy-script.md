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
- assume you have a pull secret already
- deploy a VNET with 4 subnets
  - `mgmt` will be 10.0.0.0/24
  - `external` will be 10.0.1.0/24
  - `internal` will be 10.0.2.0/**23**
  - `master-subnet` will be 10.0.4.0/**23**
- deploy ARO cluster into `master-subnet` and `worker-subnet`

This script will not: 
- complete any Azure route tables required
- complete CIS setup

### Final goal of lab build
We want to build this architecture to test out various CIS features:

<figure>
    <a href="/assets/aro/ARO-with-f5.png"><img src="/assets/aro/ARO-with-f5.png"></a>
</figure>

### az cli script

#### create VNET
```bash
LOCATION=eastus2                 # the location of your cluster
RESOURCEGROUP=oleary-aro-rg            # the name of the resource group where you want to create your cluster
CLUSTER=mycluster                 # the name of your cluster

az group create --name $RESOURCEGROUP --location $LOCATION

az network vnet create --resource-group $RESOURCEGROUP --name aro-vnet --address-prefixes 10.0.0.0/16
MGMT_SUBNET_ID=$(az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name mgmt --address-prefixes 10.0.0.0/24 | jq -r .id)
EXTERNAL_SUBNET_ID=$(az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name external --address-prefixes 10.0.1.0/24 | jq -r .id)
INTERNAL_SUBNET_ID=$(az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name internal --address-prefixes 10.0.2.0/23 | jq -r .id)
MASTER_SUBNET_ID=$(az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name aro-vnet --name master-subnet --address-prefixes 10.0.4.0/23 | jq -r .id)

```

#### deploy BIG-IP HA pair
This will deploy an ARM template via CLI: 

```bash
SSH_KEY='enter-public-key-here'
BIGIP_PASSWORD='enter-desired-password-here'
UNIQUE_STRING='enter-unique-string-here'
RESOURCE_GROUP=$RESOURCEGROUP
REGION=$LOCATION
DEPLOYMENT_NAME="parentTemplate"
TEMPLATE_URI="https://raw.githubusercontent.com/f5networks/f5-azure-arm-templates-v2/v3.3.0.0/examples/failover/azuredeploy-existing-network.json"
DEPLOY_PARAMS='{"templateBaseUrl":{"value":"https://raw.githubusercontent.com/f5networks/f5-azure-arm-templates-v2/"},"artifactLocation":{"value":"v3.3.0.0/examples/"},"uniqueString":{"value":"'$UNIQUE_STRING'"},"sshKey":{"value":"'$SSH_KEY'"},"bigIpInstanceType":{"value":"Standard_D8s_v4"},"bigIpImage":{"value":"f5-networks:f5-big-ip-best:f5-big-best-plus-hourly-25mbps:17.1.100002"},"restrictedSrcAddressMgmt":{"value":"*"},"restrictedSrcAddressApp":{"value":"*"},"bigIpRuntimeInitConfig01":{"value":"https://raw.githubusercontent.com/f5networks/f5-azure-arm-templates-v2/v3.3.0.0/examples/failover/bigip-configurations/runtime-init-conf-3nic-payg-instance01-with-app.yaml"},"bigIpRuntimeInitConfig02":{"value":"https://raw.githubusercontent.com/f5networks/f5-azure-arm-templates-v2/v3.3.0.0/examples/failover/bigip-configurations/runtime-init-conf-3nic-payg-instance02-with-app.yaml"},"useAvailabilityZones":{"value":false},"provisionPublicIpMgmt":{"value":true},"provisionServicePublicIp":{"value":true},"bigIpPasswordSecretValue":{"value":"'$BIGIP_PASSWORD'"},"bigIpMgmtSubnetId":{"value":"'$MGMT_SUBNET_ID'"},"bigIpExternalSubnetId":{"value":"'$EXTERNAL_SUBNET_ID'"},"bigIpInternalSubnetId":{"value":"'$INTERNAL_SUBNET_ID'"}}'
DEPLOY_PARAMS_FILE=deploy_params.json
echo ${DEPLOY_PARAMS} > ${DEPLOY_PARAMS_FILE}
az group create -n ${RESOURCE_GROUP} -l ${REGION}
az deployment group create --resource-group ${RESOURCE_GROUP} --name ${DEPLOYMENT_NAME} --template-uri ${TEMPLATE_URI}  --parameters @${DEPLOY_PARAMS_FILE}
```

#### deploy ARO
This will deploy ARO with imperative command:

```bash
az aro create --resource-group $RESOURCEGROUP --name $CLUSTER --vnet aro-vnet --master-subnet master-subnet --worker-subnet internal --pull-secret @pull-secret.txt
```