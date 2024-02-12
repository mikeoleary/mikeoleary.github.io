---
layout: single
title:  "Quickly deploy a BIG-IP VM in Azure"
categories: [azure]
tags: [big-ip]
excerpt: "This one is for the times I don't want to use F5's quickstart example. I want a single template (not nested templates)" #this is a custom variable meant for a short description to be displayed on home page
---
This is a simple ARM template to deploy a BIG-IP VM into an existing VNET.

#### Preparation
- have a VNET in an existing resource group deployed already, with at least 3 subnets.
- deploy with the button below:

[![Deploy to Azure](/assets/deploy-BIG-IP-ARM-template/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmikeoleary%2Ff5-bigip-standalone%2Fmain%2F%2Fbigip.json)


#### ARM template and git repo
https://github.com/mikeoleary/f5-bigip-standalone/
