---
layout: single
title:  "Migrate Bitnami Wordpress between servers"
categories: [blogging]
tags: [blogging]
excerpt: "I hate Wordpress, so I'm creating a post to remember how I migrated this site." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
As I've blogged about before, I have a friend with 2x websites that I host on Bitnami Wordpress using AWS Lightsail. For about $5 USD /mth, I get a small EC2 instance, public IP address, and Bitnami Wordpress Multi-site pre-installed on the instance. I am yet to find a cheaper way to host a Wordpress site, certainly multiple Wordpress sites, that is still relatively robust.

### Migrating 

#### Background
I have been paying about $5 USD /mth for about 4 years, running 2x websites on this single EC2 instance. In addition, I pay about $20 /yr, per domain name, for domain registration. That's $40 /yr (2 domain names) + around $60 /yr (monthly Lightsail costs), or $100 /yr that I've been paying for my friend's small business website.

He would pay me back in an instant if I asked him, but it's not just the cost that's motivating me to migrate this. I also don't want someone else's small business website running with my personal AWS account anymore. If I get hit by a bus or just plain-old forget how to manage this, the website is at risk. I would rather it exist in his AWS account. He now has a couple employees that are capable of managing this.

Time to migrate! 

Simple plan first: migrate like-for-like. Bitnami Wordpress on Amazon Lightsail, from 1 AWS account to another.

Long-term ideal plan: migrate this website away from Lightsail, on some fancy Wordpress deployment that is container-based and can run anywhere in a serverless, almost cost-free manner. I'm yet to find this ideal solution for next-to-free with almost-zero effort for migration.

#### Setting up for migration
1. I had my friend create a personal account on aws.amazon.com. He then gave me username and password.
2. I set up Lightsail instance. 

As you can see from the screenshots below, there's a few different choices for deploying an instance. You can deploy an instance with an OS only (eg Ubuntu, Windows, etc) or with an application installed (eg WordPress). I am choosing WordPress Multisite. In reality what you get is a Debian 6.1.106-3 instance that gets WordPress installed upon start up. I expect the underlying OS version and WP version to increment over time. As you can see, the lowest cost for this is currently $5 USD /mth.

<figure>
    <a href="/assets/wordpress-migration/lightsail-1.png"><img src="/assets/wordpress-migration/lightsail-1.png"></a>
    <figcaption>Lightsail EC2 Instance options</figcaption>
</figure>

<figure>
    <a href="/assets/wordpress-migration/lightsail-2.png"><img src="/assets/wordpress-migration/lightsail-2.png"></a>
    <figcaption>Lightsail pricing for Linux-based instance</figcaption>
</figure>

Broadly speaking you will follow the steps outlined here: [Set up WordPress Multisite on Lightsail](https://docs.aws.amazon.com/en_us/lightsail/latest/userguide/amazon-lightsail-quick-start-guide-wordpress-multisite.html)

#### Username, keys and passwords
Linux username is **bitnami**. This is configured by Lightsail.

When deploying, I uploaded the public key that matches the private key that I use to connect to many of my demo servers when using SSH. I will set up a password on the Debian OS when I can, and share it with my friend. I will keep my private key used for authentication in case he forgets his password. 

I may lock down the IP ranges that can access the SSH instance. I may also disable SSH access via TCP/22 completely; Lightsail allows a user to connect to an instance's command line via the Amazon web console, and I could always re-open TCP/22 if I want.

Wordpress default username is **user**. Default password is configured in a file on the instance. Run this command to retrieve it and access http://public-ip-address/wp-login.php

````bash
cat $HOME/bitnami_application_password
````

#### Setting up for migration

I ended up using a plugin to migrate my sites, and then troubleshooting when things slightly turned south. I read [this article](https://duplicator.com/wordpress-multisite-migration-plugin/) which reviewed some of the "best" plugins for migration. I didn't realize it was published by the maker of the favorite plugin until I'd finished reading.

The problem? Each one required payment for multi-site Wordpress, which I am running. I ended up coming across something called [Updraft](https://updraftplus.com/) which looked like it might do the trick with it's free option. I can't remember where I read it, but it looked like even though you are supposed to buy their premium product to get multi-site support, it may be do-able with the free version.

I installed this via the plugin marketplace on the source and destination instances. The backup on the source created 5x files, I downloaded them and uploaded them to the destination, and hit restore.

#### Migration and issues
The plugin worked, but I could not log into the site any longer. However, as far as I could tell, the content migrated okay. When I pointed my hosts file at the new server, the sites appeared to load correctly. 

But I could not log in to them. I got this message about cookies not being enabled:

<figure>
    <a href="/assets/wordpress-migration/wordpress-login-error-cookies.png"><img src="/assets/wordpress-migration/wordpress-login-error-cookies.png"></a>
    <figcaption>Annoying and misleading errors. I do have cookies enabled.</figcaption>
</figure>

##### Troubleshooting wp-admin login issues

I figured that perhaps the passwords of the users did not get migrated correctly, so I followed [this link](https://docs.bitnami.com/aws/apps/wordpress/administration/reset-wp-admin-password/) to reset the passwords. 

However, that wasn't the problem. I still could not log in. Then I found [this page](https://wp-staging.com/how-to-fix-the-error-cookies-are-blocked-or-not-supported-by-your-browser/) that gave me a few ideas:

###### Disable plugins

I followed the first step and renamed `/wp-content/plugins/` to `/wp-content/plugins.hold` and then restarted apache. I _think_ this is supposed to disable plugins. Anyway, I still got the error from the screenshot above, so I renamed it back and moved on. This particular troubleshooting step was also suggested [here](https://wordpress.org/documentation/article/faq-troubleshooting/#how-to-deactivate-all-plugins-when-not-able-to-access-the-administrative-menus).

###### Editing wp-config.php
I then followed the suggestion of adding this line to my wp-config.php file:
`define('COOKIE_DOMAIN', $_SERVER['HTTP_HOST'] );`

I also restarted apache. That seemed to do it! I can now log into the admin console.

##### Troubleshooting SSL issues
The plugin did not move my SSL certificates. I just manually grabbed them off the old server at `/opt/bitnami/apache2/conf` and then copied the contents of the cert and key. Then I just overwrote the files on the new server at `/opt/bitnami/apache/conf/bitnami/certs/server.crt` and `/opt/bitnami/apache/conf/bitnami/certs/server.key`

The reason I did this was because I didn't want to cut public DNS over to the new servers just yet, so I couldn't use the cert tool with Let's Encrypt. (I will go back and do this later. Good thing I [documented it last time]({% post_url 2024-08-15-lets-encrypt-for-bitnami-wordpress %})).

Long story short on TLS certs: every IT person needs to know the basics of TLS, handshakes, the cert tool and LetsEncrypt.

##### Network Admin dashboard
Finally, I think I broke something because the Network Admin dashboard would not work. Here's a screenshot of what it would look like when I wanted to look at the dashboard for ALL sites (not just 1 of my 2 sites).

<figure>
    <a href="/assets/wordpress-migration/network-admin-dashboard-error.png"><img src="/assets/wordpress-migration/network-admin-dashboard-error.png"></a>
    <figcaption>What happens here is the dashboard is trying to load a page at http://publicIP.nip.io which I don't control.</figcaption>
</figure>

After googling the phrase "wordpress change hostname of network admin dashboard" - because I suspected that the hostname that shows up in the screenshot would need to be updated in settings or the DB - I found these 2 links [here](https://stackoverflow.com/questions/40812224/wordpress-changed-hostname-cant-access-admin) and [here](https://stackoverflow.com/questions/18336538/wordpress-wp-siteurl-and-wp-home-values).

I _think_ I can set the values of `WP_HOME` and `WP_SITEURL` in the wp-config.php file, OR the values of `home` and `siteurl` in the `wp_options` table in the DB. I liked the idea of doing it in the DB. So I made these mysql commands:

````bash
$ mysql -u root -p bitnami_wordpress -e "SELECT * FROM wp_options WHERE option_name = 'siteurl';"
mysql: Deprecated program name. It will be removed in a future release, use '/opt/bitnami/mariadb/bin/mariadb' instead
Enter password:
+-----------+-------------+-----------------------------+----------+
| option_id | option_name | option_value                | autoload |
+-----------+-------------+-----------------------------+----------+
|         1 | siteurl     | https://x.x.x.x.nip.io | yes      |
+-----------+-------------+-----------------------------+----------+
$ mysql -u root -p bitnami_wordpress -e "SELECT * FROM wp_options WHERE option_name = 'home';"
mysql: Deprecated program name. It will be removed in a future release, use '/opt/bitnami/mariadb/bin/mariadb' instead
Enter password:
+-----------+-------------+-----------------------------+----------+
| option_id | option_name | option_value                | autoload |
+-----------+-------------+-----------------------------+----------+
|         2 | home        | https://x.x.x.x.nip.io | yes      |
+-----------+-------------+-----------------------------+----------+

````

Then I saw the URL that would not load, so I updated the values:
````bash
mysql -u root -p bitnami_wordpress -e "UPDATE wp_options SET option_value = 'https://my-preferred-fqdn.com' WHERE option_name = 'home' OR option_name = 'siteurl';
````

This worked and updated the DB values, but didn't seem to allow me to log in. So I updated the wp-config.php file with this:

````
define('WP_HOME','https://my-preferred-fqdn.com')
define('WP_SITEURL','https://my-preferred-fqdn.com')
````

Restarted apache, but still no luck. I'm giving up here. I will live with no Network Admin dashboard for now.


#### Tips

Like every time I deal with Wordpress, I got lost in version mis-matches, poor documentaton, a wild west of plugins and gotchas, like plugins that require a monthly fee _charged annually_ to do something you may need just one time.

So here's some general tips I will make note of, since I can't seem to get a smooth and reliable procedural document written:

- restart apace with this command:
`sudo /opt/bitnami/ctlscript.sh restart apache`

- reset wp-admin password:
```bash
mysql -u root -p bitnami_wordpress -e "SELECT * FROM wp_users;"
mysql -u root -p bitnami_wordpress -e "UPDATE wp_users SET user_pass=MD5('NEWPASSWORD') WHERE ID='ADMIN-ID';"
```

- location of wp-config.php file. I never seem to know where it is. On my new server, it is here:
`/bitnami/wordpress/wp-config.php` and there is also a symlink at `/opt/bitnami/wordpress/wp-config.php`