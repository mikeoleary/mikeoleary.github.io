---
layout: single
title:  "Quickly create openssl cert and key"
categories: [openssl]
tags: [openssl]
excerpt: "I can never remember this one-liner so here it is to bookmark." #this is a custom variable meant for a short description to be displayed on home page
---
Here's a one-liner for generating a self-signed certificate. This is primarily for me to quickly find when I cannot remember the openssl command line.

### 
- key length is 2048
- requires openssl
- cert validity is 365 days

````
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout server.key -out server.crt
````


