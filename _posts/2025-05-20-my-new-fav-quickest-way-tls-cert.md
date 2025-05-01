---
layout: single
title:  "My fastest way to request public TLS certs"
categories: [docker]
tags: [docker,aws]
excerpt: "Quick notes for future use" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/quick-certbot-request/quick-certbot-request.png"><img src="/assets/quick-certbot-request/quick-certbot-request.png"></a>
</figure>

### Summary
Quick command to run on Docker server to use certbot to request public TLS cert, using DNS as the ACME protocol challenge method.

### Why use this command
- you don't want to install certbot for some reason
- you want to use a container because it's quick and you can delete it when done
- this is a once-off cert request (if you want to schedule regular renewal, take this and research further)

### Prerequisites
- you need a AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY. If they are temporary creds, then you also need the AWS_SESSION_TOKEN.
- in that AWS account, you need a public hosted zone that matches your requested domain.
  - If your DNS domain was registered outside of this AWS account, make sure your NS servers in your domain registrar point to the NS records in this hosted zone. Ie, you must own the DNS domain and have it set up correctly in order Let's Encrypte to complete the challenge.

### Docker command to run
````bash
$ docker run --rm -it --name certbot \
--env AWS_ACCESS_KEY_ID=ASIAxxxxxxxxxxxxxxxx \
--env AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
--env AWS_SESSION_TOKEN="xxx.............xxx" \
-v "/etc/letsencrypt:/etc/letsencrypt" \
-v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
certbot/dns-route53 certonly \
--dns-route53 \
-d helloworld.example.com \
--agree-tos

````
#### Let's break that down
- `--rm` just tells Docker to remove the container and image when done
- `-it` tells Docker to show console output while it runs
- `--env` provides env vars to the container, we're passing our AWS temp programmatic creds
- `certbot/dns-route53` is the container image, this is certbot with the dns-route53 plugin installed
- `certonly` the first parameter we are passing to the certbot command, because we just need the certs (we don't want certbot to touch web servers)
- `--dns-route53` this tells certbot to use the Route 53 plugin for the DNS challenge
- `-d` for domain requested
- `--agree-tos` agrees to ACME Subscriber Agreement

### Conclusion
Run this command to quickly request a Let's Encrypt cert without needing to install certbot. All you need is your valid domain set up in Route53 and temp programmatic creds.