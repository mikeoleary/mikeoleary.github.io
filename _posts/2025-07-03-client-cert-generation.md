---
layout: single
title:  "Quickly generate TLS client cert (self-signed)"
categories: ["tls"]
tags: ["tls"]
excerpt: "Recently I was asked to create TLS client certs for mTLS testing. Here's a quick script" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/client-cert-creation/client-cert-header.png"><img src="/assets/client-cert-creation/client-cert-header.png"></a>
</figure>

### Summary
Just like my [recent post]({% post_url 2025-05-20-my-new-fav-quickest-way-tls-cert %}) with a 1-liner for TLS cert creation, this is simply for me to copy/paste later.

### Client cert creation for mTLS
```bash
# Create the CA key and cert
openssl genrsa -aes256 -out ca.key 4096
openssl req -new -x509 -sha256 -days 365 -key ca.key -out ca.crt -subj "/CN=MyRootCA"

# Create the Client Certificate:
## Generate the client private key. 
openssl genrsa -out client.key 2048
## Create the client CSR
openssl req -new -key client.key -out client.csr -subj "/CN=myclient"
## Sign the client CSR
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256

# Verify the client certificate. 
openssl verify -CAfile ca.crt client.crt
```

### To configure with .cnf files
Here's a nice little [demo bash script](https://github.com/SalesAmerSP/mtls) I became aware of after running the commands. Use this if you want to configure things like Common Names and other attributes in config files.