---
layout: single
title:  "Let's Encrypt cert automation for Bitnami Wordpress"
categories: [blogging]
tags: [blogging]
excerpt: "I will forget if I don't document this. How to configure SSL certs for Bitnami Wordpress." #this is a custom variable meant for a short description to be displayed on home page
gallery:
  - image_path: /assets/wordpress-letsencrypt/henkle-cosmetics.png
    url: /assets/wordpress-letsencrypt/henkle-cosmetics.png
    title: "Henkle Cosmetics"
  - image_path: /assets/wordpress-letsencrypt/lowell-pain-center.png
    url: /assets/wordpress-letsencrypt/lowell-pain-center.png
    title: "Lowell Pain Center"
---
### Summary
I am no Wordpress expert. I have found a Bitnami Wordpress image, running via AWS Lightsail, to be very cheap. I pay around $6/mth for a small EC2 instance, VPC, public IP address, all as part of a Lightsail deployment.

### How to update SSL certs for Bitnami Wordpres
1. Log into AWS with personal account.
  - username is my personal email address
  - password is stored in vault
2. Navigate to Lightsail service, and see the public IP of the EC2 VM instance that's running.
3. SSH to the VM. 
  - Username is **bitnami**.
  - SSH key: **mikeo-keypair-personal.ppk**
4. Run `sudo /opt/bitnami/bncert-tool`
5. I had to then fill out the prompts with the URL's that I wanted, comma-separated.
6. That should be it. After this, there should be a cron job that updates the certs every 90 days or so.

You can also [manually use the cert tool to generate SSL certs](https://docs.bitnami.com/aws/how-to/generate-install-lets-encrypt-ssl/#alternative-approach) and then put them in the correct directory.

<figure>
    <a href="/assets/wordpress-letsencrypt/cli-screen-capture.png"><img src="/assets/wordpress-letsencrypt/cli-screen-capture.png"></a>
    <figcaption>Logging in via SSH to public IP of my EC2 instance</figcaption>
</figure>

### Wordpress Admin
This is something I have left up to the website owner, but for my own reference:
- URL: https://the-domain-of-the-pain-clinic/wp-admin
- username: henklecosmetics
- password: [this one has been communicated to website admin]

There is another user, too:
- URL: https://the-domain-of-the-pain-clinic/wp-admin
- username: my personal email address
- password: [this one has been communicated to website admin]

### Documentation
This worked for me: https://docs.bitnami.com/aws/how-to/generate-install-lets-encrypt-ssl/

### Screenshots
{% include gallery id="gallery" caption="Screenshots of websites. No TLS warnings!"  %}
