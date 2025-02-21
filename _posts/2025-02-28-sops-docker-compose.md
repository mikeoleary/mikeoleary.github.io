---
layout: single
title:  "Using SOPS with Docker Compose"
categories: [docker]
tags: [docker]
excerpt: "Managing secrets with Docker Compose leaves us a few options, and a sidecar container is one approach" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

<figure>
    <a href="/assets/sops-docker-compose/sops_decryption_v2.png"><img src="/assets/sops-docker-compose/sops_decryption_v2.png"></a>
</figure>

### Summary
Docker Compose offers a few options for dealing with secret management. In this post I'll show one approach, which is to use SOPS in a "sidecar container" to provide a secret to an application container.

### My use case
Recently I had a colleague reach out and say 
> Hey Mike, I have a Docker Compose file to stand up 3 containers. One of the containers has a config file mounted and that file contains passwords. How can we handle this correctly?

The answer varies on your desired complexity, but the basic idea is to keep passwords out of code, and out of the configuration files committed to source control.

#### Some dead ends
First, some dead ends.

Consistently, [this article](https://blog.gitguardian.com/how-to-handle-secrets-in-docker/) showed up in Google searches. It lists 4 ways to use Secrets when faced with Docker. I've listed them, along with my thoughts on each:
1. **Docker Secrets (using Swarm)**. I don't want to use Swarm, so this is out.
2. **Docker Compose secrets (using plaintext file).** I didn't like plaintext files, even if they were kept out of source control.
3. **Sidecar container.** This is the approach I used with SOPS.
4. **Using Mozilla SOPS.** As far as I can tell, this is still a sidecar container approach. 

I have a few issues with this article. 

Firstly, I'm pretty sure there's some errors in the example configs, where a secret may be called `my_secret` in one place but `my_secret_key` in another place. Typos are part of life, but this article has too many errors.

My big issue, however, is that Options 3 and 4 are the same thing. SOPS can be used to decrypt a secret; it is not a vault itself. Regardless, SOPS acts as a "sidecar container" that provides a secret to the application container. And the [example config](https://blog.gitguardian.com/how-to-handle-secrets-in-docker/#using-mozilla-sops) for Option 4 showing SOPS with docker compose just doesn't work. 

<figure>
    <a href="/assets/sops-docker-compose/sops-article-screenshot.png"><img src="/assets/sops-docker-compose/sops-article-screenshot.png"></a>
    <figcaption>Alas, 3rd party blogs sometimes leave you confused. Official docs on <a href="https://docs.docker.com/compose/how-tos/use-secrets/">Docker Compose Secrets</a> are more reliable. Please reach out to me if you're ever confused about anything I write. </figcaption>
</figure>

I'm pretty sure there's multiple reasons the config pictured above won't work:
- there's nothing shared (no network, no volume) between the SOPS container and the app container
- SOPS doesn't magically create environment variables for other containers
- Docker Compose secrets with `external: true` refers to secrets external to Docker Compose. These must come from a plaintext file or Docker Swarm. 

### SOPS as a sidecar container
#### This isn't perfect
First off, using SOPS as a sidecar container isn't perfect. Some obvious issues with this set up:
- The shared volume between the sidecar container and the application container is still, under the hood, somewhere on the Docker host, obviously. So if it contains an unencrypted secret, that secret is on the Docker host.
- In this example I'm passing an encrypted secret and the private key with which to decrypt. Obviously the private key shouldnt be committed to a repo. The idea here is that this can only be run from a Docker host that has access to that private key.

It might be more feasible to install the SOPS utility in the container image of the application container itself. Then have the application container decrypt and use the secret itself, no sidecar required. But in my case, we did not have the option to edit the application container at all, so the sidecar approach at least lets us 

#### How to set up SOPS as a sidecar container
Let's take a look at this Docker Compose file. I have a local directory called `secrets` with 2 files: `private.rsa` and `secret.enc.txt` which is an encrypted file that I have created before this demo.

```bash
version: '3'

volumes:
  sharedsecretvol:

services:
  sops:
    image: mozilla/sops:latest
    container_name: sops
    command:
       - /bin/sh
       - -c
       - |
         gpg --import /secrets/private.rsa
         sops --config /secrets/.sops.yaml --decrypt /secrets/secret1.enc.txt > /sharedsecretvol/secret1.txt
    volumes:
      - ./secrets:/secrets
      - sharedsecretvol:/sharedsecretvol

  busybox:
    image: busybox
    container_name: busybox
    restart: "no"
    volumes:
      - sharedsecretvol:/sharedsecretvol
    command:
      - /bin/sh
      - -c
      - cat /sharedsecretvol/secret1.txt
    depends_on:
      sops:
        condition: service_completed_successfully
```

Here's the order of events when we use SOPS as a sidecar container
1. Docker Compose is run (`docker-compose up`).
  a. A volume is mounted to the SOPS container that contains an encrypted secret and a private key
  b. A volume will be mounted that is shared between the 2 containers.
2. The SOPS container is run first.
  a. The SOPS command imports the private key, decrypts the secret, and then outputs a clear-text file to a shared volume.
3. After the SOPS container is run, the Application container is run.
  a. We use the [depends_on](https://docs.docker.com/reference/compose-file/services/#depends_on) attribute to control the order of service startup and shutdown because our application requires the file to be present at startup.
  b. The application can now run and access the secret required.

Here is our output when we run `docker-compose up`. Take a look at line 6. This is the contents of the file `secrets.txt` which was decrypted from `secrets.enc.txt` and provided to our busybox container.

```bash
azureuser@docker-demo1:~$ docker-compose up
Creating network "azureuser_default" with the default driver
Creating sops ... done
Creating busybox ... done
Attaching to sops, busybox
busybox    | YOUR-PASSWORD-HERE
busybox    |
sops       | gpg: directory '/root/.gnupg' created
sops       | gpg: keybox '/root/.gnupg/pubring.kbx' created
sops       | gpg: /root/.gnupg/trustdb.gpg: trustdb created
sops       | gpg: key 5CBF3196AF775ABF: public key "Michael O'Leary <mi.oleary@f5.com>" imported
sops       | gpg: key 5CBF3196AF775ABF: secret key imported
sops       | gpg: Total number processed: 1
sops       | gpg:               imported: 1
sops       | gpg:       secret keys read: 1
sops       | gpg:   secret keys imported: 1
sops exited with code 0
busybox exited with code 0
```

### Conclusion
Secrets management is always important. I like to think of SOPS as a handy utility for editing and encrypting/decrypting, and the use of SOPS as a sidecar container is something that can be done relatively easily. Thanks for reading!

