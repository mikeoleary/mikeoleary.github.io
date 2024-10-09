---
layout: single
title:  "Migrate Bitnami Wordpress between servers, Part 3"
categories: [blogging]
tags: [blogging]
excerpt: "I hate Wordpress, so I'm creating a post to remember how I migrated this site." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
Ugh. I thought I was done, but I have a Part 3 to write.

[Part 1]({% post_url 2024-10-12-migrate-bitnami-wordpress-between-servers %})<br>
[Part 2]({% post_url 2024-10-13-migrate-bitnami-wordpress-between-servers-part-2 %})

### AWS Lightsail Bitnami Wordpress and sending mail
I realized that I had set up Amazon SES in the past under my own account. I needed to move it to my buddy's AWS account. Here we go.

Short version is that using Amazon SES is [documented here](https://docs.bitnami.com/aws/how-to/use-ses/), but really that's no better than generic Amazon SES documentation. We want something Wordpress-specific, and ideally something that is "Bitnami Wordpress on AWS Lightsail"-specific.

Within Wordpress, I actually had to activate the plugin called WP Mail SMTP, just like is outlined in these [specific instructions](https://docs.bitnami.com/aws/apps/wordpress/configuration/configure-smtp/) from Bitnami. 

I would take screenshots, but those in the linked document above were exactly what I had. After following the first link and creating SMTP creds in SES, I was able to enter them into my Wordpress installation, and configure the SMTP host `email-smtp.us-east-2.amazonaws.com`.

After verifying the email address I want to send my mail to, and the domain, my contact forms now work.

After doing all of this, I see that Amazon has [instructions](https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-enabling-email-on-wordpress.html) specifically for Lightsail Wordpress instances. These are the best instructions that I've seen regarding this procedure.

### A couple more things

- I changed my wp-config.php file one last time:

`define( 'DOMAIN_CURRENT_SITE', 'lowellpaincenter.com' );`<br>
was changed to<br>
`define( 'DOMAIN_CURRENT_SITE', $_SERVER['HTTP_HOST'] );`

I think this allows me to login to either site and get to the Network Admin Dashboard. Before, I could only do this if logged into lowellpaincenter

- I removed SSH (TCP/22) from allowed FW rules in Lightsail. Now, for admin access to CLI, myself or someone else will need to log into the AWS Lightsail console and re-enable. Even after that, SSH will require authenticating with my private key.



