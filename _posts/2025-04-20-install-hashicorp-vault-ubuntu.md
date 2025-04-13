---
layout: single
title:  "My Vault Setup"
categories: [docker]
tags: [docker, vault]
excerpt: "This is a record so I can reference how I like to run Hashicorp Vault" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/vault-setup/vault-header-image.png"><img src="/assets/vault-setup/vault-header-image.png"></a>
</figure>

### Summary
Here's a record in case I need to re-deploy Vault.

### Why am I deploying Vault?
After a few recent posts pondering whether I should use [SOPS or Vault Agent]({% post_url 2025-04-05-using-vault-agent-and-sops %}), and how I might inject secrets with SOPS running [locally]({% post_url 2025-03-28-using-sops-locally-with-docker %}) or in a [sidecar container]({% post_url 2025-02-28-sops-docker-compose %}), I think I'm leaning toward a KMS-backed Vault installation.

The [key features](https://developer.hashicorp.com/vault/docs/what-is-vault#why-vault) of Vault are just too good, even when I weigh up potential risks of secret management (ie using Vault) vs keeping secrets encrypted via a KMS (ie using SOPS).

This [matrix](/assets/vault-docker-compose/agent-vs-sops-matrix.png) I used in my earlier post was helpful for me. I thought that I could still use Vault as a encryption/decryption tool backed by a KMS, similar to SOPS. However, when digging into this more, I think that a more important question than *"Vault Agent vs SOPS"* would be *"Vault Transit vs SOPS for stateless encryption/decryption"*. 

I.e., is it ever safe to commit encrypted data in Git? What's the difference if you use Vault or SOPS? (You're allowed to use AI to answer.)

#### Sidenote post: a ChatGPT-generated post
Check out this [accompanying blog post]({% post_url 2025-04-20-vault-transit-vs-sops-for-git-commit %}) to answer the question above. My take: if you have access to the encryption key, then no, it's not safe to commit encrypted secrets to git, because you could leak the key. 
- SOPS relies on a KMS with a DEK and KEK, so we can assume that key is safe and you can committ encrypted secrets.
- Vault Transit is *almost* as safe. It doesn't allow anyone - not even a Vault administrator - access to the encryption keys, but if an admin had root access to the OS on which Vault ran, then they *could* get this key with a fair amount of work. (Ask ChatGPT how). So I would avoid committing encrypted data in this case.

To keep us on track though, Vault Transit is similar to SOPS and since I won't be committing encrypted secrets anyway, and am more interested in the operations of sophisticated and efficient secret management, I'm proceeding with Vault.

### How am I running Vault?
My trusty workstation runs VM's that I want to keep stateful and not incur cloud costs (you're welcome, F5!).

Here's my installation of Vault on Ubuntu:

{%highlight bash%}
#Install dependencies
sudo apt update && sudo apt install -y gnupg curl unzip

#Add HashiCorp GPG key
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

#Add official repo
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

#Install Vault
sudo apt update && sudo apt install -y vault

#Confirm version
vault --version
{% endhighlight%}

### Next post: using Vault
OK, after all of this, my next post will be what I've been meaning to get to: using Vault Agent to populate secrets for Docker containers. We'll start with non-Kubernetes and maybe cover Kubernetes options as well.