---
layout: single
title:  "Migrate Bitnami Wordpress between servers, Part 2"
categories: [blogging]
tags: [blogging]
excerpt: "I hate Wordpress, so I'm creating a post to remember how I migrated this site." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
Yesterday I [wrote a blog post]({% post_url 2024-10-12-migrate-bitnami-wordpress-between-servers %}) about migrating my multi-site Wordpress deployment between 2 servers. It ended in failure because I could not reach the Network Admin dashboard, even though my 2 sites were accessible.

This evening I blew away the new server (twice, in the end) and tried again and worked it out. Here's a record so I remember the changes that I made.

### Migrating to a new server
1. Set up new AWS Lightsail server.
2. Change it so that it has a static IP address that persists across reboots
3. Ensure the private key for SSH access is as desired.
4. Deploy, and wait a few mins for Wordpress multi-site deployment to finish.
5. Install only the plugin called Updraft
6. Restore the backup that you created on source server, using Updraft, to new server.
7. When restoring, do not restore plugins. I don't know if they cause a problem, but to rule it out, I did not migrate the plugins.
8. After restoration, I had this situation
- www.henklecosmetics.com was accessible at the expected URL without problems
- lowellpaincenter.com was not accessible. This site was called "52.2.160.149.nip.io"

### Fixing the site name

#### Database updates
I believe there are only a few DB tables I need to care about. They are
- wp_blogs
- wp_site
- wp_options
- wp_%_options

Now, in the end I only edited 2 tables. 
- `wp_site` had only 1 entry. I left this alone.
- `wp_blogs` had 3 entries, each with the same site_id but different blog_id. It was straightforward to see which one needed updating from `52.2.160.149.nip.io` to `lowellpaincenter.com`
- `wp_options` appeared to be the options table for the "main" site. I think that's a default site. You can't delete that site, so I left this table alone.
- `wp_6_options` was the table that I updated. The `6` corresponds to the `blog_id` from the wp_blogs table. I updated the values for `home` and `siteurl`.

#### Config file updates

I believe I ended up making only 2x changes to the wp-config.php file. They were
1. `define( 'DOMAIN_CURRENT_SITE', '52.2.160.149.nip.io' );` was changed to `define( 'DOMAIN_CURRENT_SITE', 'lowellpaincenter.com' );`
2. `define( 'COOKIE_DOMAIN', $_SERVER['HTTP_HOST'] );` was added. This fixed the cookie error on login if I was trying to log in to admin panel from henklecosmetics and not from lowellpaincenter.

Restarted apache and things seemed to work.

### Redirect rules
Around Sept 11, 2024, I updated the "old" server to accept any of the 4 variations of domain names: 
- henklecosmetics.com (which should redirect to www.henklecosmetics.com)
- www.henklecosmetics.com
- lowellpaincenter.com
- www.lowellpaincenter.com (which should redirect browsers with a 301 or 302 to just lowellpaincenter.com)

All of these should work, whether HTTP or HTTPS, because my certificate is valid for all 4 domain names. Don't ask me why one of the sites uses "www" and the other does not. I really don't remember that.

In any case, I googled and learned how to do it with my new server, following the [official documentation](https://docs.bitnami.com/aws/apps/wordpress/administration/redirect-custom-domains/) from Bitnami Wordpress packaged for AWS.

I followed these steps and created some files in this location: `/opt/bitnami/apache/conf/vhosts`

### Other fixes
- I still had to copy over the old TLS certs again, as outlined in the previous blog post. 
- I will still run the bncert tool in the future to ensure that future Lets Encrypt certs are deployed.

### Conclusion
At this point, it looks like I've successfully migrated my Wordpress multi-site deployment between AWS Lightsail servers. I've managed to do this without paying for a commercial migration plugin that supports multi-site, but it did take me 2x late nights of troubleshooting Wordpress.

At least I've documented a handful of things in case I need to do this again. More importantly, the AWS account is owned by the website owner, so I can get this off my credit card.


