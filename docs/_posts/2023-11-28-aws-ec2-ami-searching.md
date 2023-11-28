---
layout: post
title:  "Quickly search for F5 images in Azure and AWS"
categories: [aws, azure]
tags: [aws, azure]
description: "This post contains commands for searching the AWS and Azure VM image marketplaces, so that I can quickly copy and paste next time." #this is a custom variable meant for a short description to be displayed on home page
---
<!-- begin_excerpt -->
This post is solely so I can copy/paste this in future instead of spending 5 mins re-learning this every 6 months.

<!-- end_excerpt -->
### Azure
{% highlight bash %}
    #commands to search f5 images in Azure Marketplace with az cli

    #get all offers from f5
    az vm image list --publisher f5-networks --location eastus2 --all | jq .[].offer

    #get all skus within an offer
    az vm image list --publisher f5-networks --offer f5-big-ip-best --location eastus2 --all | jq .[].sku

    #get all versions within a sku
    az vm image list --publisher f5-networks --offer f5-big-ip-best --sku f5-bigip-virtual-edition-200m-best-hourly --location eastus2 --all | jq .[].version

    #get a specific image
    az vm image list --publisher f5-networks --offer f5-big-ip-best --sku f5-bigip-virtual-edition-200m-best-hourly --location eastus2 --all

{% endhighlight %}

### AWS
{% highlight bash %}
    #commands to search f5 AWS AMI's using aws cli

    #filter images by description
    aws ec2 describe-images --region us-east-1 --filter "Name=name,Values=F5*BIGIP*16.1*" | jq .Images[].Name

{% endhighlight %}
