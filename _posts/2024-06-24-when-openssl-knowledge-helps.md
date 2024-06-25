---
layout: single
title:  "When OpenSSL knowledge helps"
categories: [openssl]
tags: [openssl]
excerpt: "This week I wasted time unnecessarily converting certificate and key file formats, and this post explains why." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<!--
<figure>
    <a href="/assets/new-laptop/new-laptop-setup.jpg"><img src="/assets/new-laptop/new-laptop-setup.jpg"></a>
    <figcaption>My new laptop. For cleaner looking presentations, I never save any files to my desktop.</figcaption>
</figure>
-->
#### Background
Last week, during a heat wave in June 2024, I traveled to the Southeast for a potential customer. Our plan was ambitious: to cut over from one technology to another, around midnight, and quickly troubleshoot and fix any unforeseen technical troubles as they arose. 

During the week I was forced to quickly write a script containing openssl commands to convert PFX files to cert and key files, and then import them into BIG-IP. I learned 2 big lessons that I will summarize this post with.

<!--
<figure>
    <a href="/assets/new-laptop/old-laptop1.jpg"><img src="/assets/new-laptop/old-laptop1.jpg"></a>
    <figcaption>Some engineers enjoy tinkering with desktop hardware and would attempt to fix this. These days I can't prioritize that and had it replaced.</figcaption>
</figure>
-->
#### The challenge

Upon arriving at the site, I realized that my configurations - the configuration I planned to load into the new technology, had hundreds of references to hundreds of different SSL certificates and private keys. One problem - the configurations I had referenced placeholder certs/keys. 

The actual certs/keys belonged to the customer and had different names. I would have no access to the cert or key material. I had to update my configuration file to reference certs and keys that I didn't actually have.

#### My plan to address this

Fairly quickly, I came up with a plan. Without going into too much detail, I made an Excel spreadsheet based on the previous technology's configuration. I mapped every cert and key from the previous config to virtual servers in the previous config. The names of virtual servers in my new config were based on those on the old config. In this way, I had a mapping of certs and keys from old config to certs and keys that *should* be in my new config.

##### But what about cert / key material?
Since I couldn't get access to cert/key material, I had to have the customer install certificates prior to loading my new config, and then reference the location of these in my new configuration. But they would need help. I'd need to write a script to have them do this. 

Oh - and they didn't know the passphrase of their certificates, but they could provide me with PFX files of their certificates. Thank God, the PFX files had the same name as their references in the old config. I needed to:

 - write a script to install about 423 certs and keys in PEM format to BIG-IP
 - write a script to install 111 PFX files to BIG-IP

##### Dumb thing I did, #1
I read [K14031](https://my.f5.com/manage/s/article/K14031) and [K14912](https://my.f5.com/manage/s/article/K14912) and for some reason concluded that importing PFX files directly using TMSH wasn't possible (it is). So I wrote a script to do this:

````bash
# Break apart 111 PFX files into certs and keys
## Save cert as PEM file certificate
openssl pkcs12 -in pfx-file-name.pfx -nokeys -clcerts -out pfx-file-name.pem -passin pass:'XXX'
## Save key as PEM file that is encrypted
openssl pkcs12 -in pfx-file-name.pfx -nocerts -out pfx-file-name.key_encrypted -passin pass:'XXX' -passout pass:'XXX'
## Now, remove the passphrase from the key file and output a clear text key
openssl rsa -in pfx-file-name.key_encrypted -out pfx-file-name.key -passin pass:'XXX'
# Install 111 certs and 111 keys into the BIG-IP
tmsh install /sys crypto cert pfx-file-name.pem from-local-file pfx-file-name.pem
tmsh install /sys crypto key pfx-file-name.pem from-local-file pfx-file-name.key
````

... and I looped through 111 files. However, after reading [K6549](https://my.f5.com/manage/s/article/K6549) and [this reference](https://clouddocs.f5.com/cli/tmsh-reference/v14/modules/sys/sys_crypto_pkcs12.html) I learned this:

>starting in BIG-IP 11.0.0, you can also import PKCS#12 files directly using the TMOS Shell (tmsh)

So what I should have done was this, which should have saved me the redundant effort of creating separate .pem and .key files:

````bash
tmsh install pkcs12 pfx-file-name passphrase 'XXX' from-local-file '/tmp/pfx-file-name.pfx'
````

##### Dumb thing I did, #2
The first time I ran my script above, I did not use the `-clcerts` flag. [This flag](https://www.openssl.org/docs/man1.0.2/man1/pkcs12.html) instructs openssl to extract the child certificate only, not intermediate or root certs if they are present. I found that if I left this flag off, the relationship between child, intermediate, and root cert would be 'backward' when I imported into BIG-IP. 

What do I mean by 'backward'? I mean that when I imported the .pem file I created, I could not import the key because BIG-IP complained that cert and key didn't match. Also, the TMUI console (BIG-IP GUI) showed the certificates in the opposite order as desired. Long story short, I didn't need to install the Intermediate and Root certs, so removing these from my .pem file allowed my script to work.

#### Conclusion and lessons learned

1. TMSH can import .PFX files directly
2. It really helps to know openssl commands and be familiar. I don't use the utility very often, but when I do, I can't really afford to learn from the ground up. Basic adequacy in SSL concepts and terminology is a minimum requirement today, I believe.

Thanks for reading :)