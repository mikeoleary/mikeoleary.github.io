---
layout: single
title:  "Quickly get your kubeconfig file"
categories: [kubernetes]
tags: [kubernetes, azure, aws, aks, eks, gke]
excerpt: "Quick notes to copy and grab your kubeconfig file when using hosted clusters." #this is a custom variable meant for a short description to be displayed on home page
---
<!-- begin_excerpt -->
Once in a while I need to download or generate a kubeconfig file for a cluster, usually when it's in AKS and I have forgotten, deleted, or otherwise lost my kubeconfig file. Here's some quick notes so I can copy and paste this later.

<!-- end_excerpt -->
### Azure Kubernetes Service (AKS)
{% highlight powershell %}
## output to a file with the --file argument, otherwise it merges with the default kubeconfig
$ResourceGroup = 'my-resource-group-name'
$ClusterName = 'my-cluster-name'
$FileName = 'aks-kubeconfig'
az aks get-credentials --resource-group $ResourceGroup --name $ClusterName --file $FileName
{% endhighlight %}

### Amazon Elastic Kubernetes Service (EKS)

{% highlight bash %}
## use the --kubeconfig argument to merge with a file other than your default kubeconfig file
## https://docs.aws.amazon.com/cli/latest/reference/eks/update-kubeconfig.html

REGIONCODE='us-east-1'
CLUSTERNAME='myekscluster'
aws eks update-kubeconfig --region $REGIONCODE --name $CLUSTERNAME
{% endhighlight %}

### Google Kubernetes Engine (GKE) 

{% highlight bash %}
## https://cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials
## For zonal clusters, use --zone=COMPUTE_ZONE instead of --region

REGION='us-central1'
PROJECTNAME='my-gcp-project'
CLUSTERNAME='my-eks-cluster'
gcloud container clusters get-credentials $CLUSTERNAME --project $PROJECTNAME --region $REGION
{% endhighlight %}

