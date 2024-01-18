---
layout: single
title:  "Quickly deploy a Palo Alto VM in Azure with this ARM template"
categories: [azure]
tags: [palo-alto]
excerpt: "After too many one-off ARM templates, I'm documenting a simple and reusable ARM template for myself. Use this when deploying a PA firewall in Azure." #this is a custom variable meant for a short description to be displayed on home page
---
This is a simple ARM template to deploy a Palo Alto VM into an existing VNET.

#### Preparation
- have a VNET in an existing resource group deployed already, with at least 3 subnets.
- deploy with the button below:

[![Deploy to Azure](http://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fgist.githubusercontent.com%2Fmikeoleary%2Fbb5e7d2e5baafa2eccd59e3430b194cb%2Fraw%2Fgistfile1.txt)


#### ARM template
<script src="https://gist.github.com/mikeoleary/bb5e7d2e5baafa2eccd59e3430b194cb.js"></script>
