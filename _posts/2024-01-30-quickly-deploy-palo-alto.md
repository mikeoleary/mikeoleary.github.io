---
layout: single
title:  "Quickly deploy a Palo Alto VM in Azure with this ARM template"
categories: [azure]
tags: [palo-alto]
excerpt: "After too many one-off ARM templates, I'm documenting a simple and reusable ARM template for myself. Use this when deploying a PA firewall in Azure." #this is a custom variable meant for a short description to be displayed on home page
---
This is a simple ARM template to deploy a Palo Alto VM into an existing VNET.

### ARM template

- have a VNET in an existing resource group deployed already, with at least 3 subnets.

<script src="https://gist.github.com/mikeoleary/bb5e7d2e5baafa2eccd59e3430b194cb.js"></script>
