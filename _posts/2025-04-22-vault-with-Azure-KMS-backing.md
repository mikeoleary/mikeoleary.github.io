---
layout: single
title:  "Vault with Azure Key Vault backing"
categories: [docker]
tags: [docker, vault]
excerpt: "Continuing documenting Hashicorp Vault so I can recreate later" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/vault-setup-kms/vault-kms-header-image.png"><img src="/assets/vault-setup-kms/vault-kms-header-image.png"></a>
</figure>

### Summary
Vault was installed in the [last post]({% post_url 2025-04-20-install-hashicorp-vault-ubuntu %}). Now let's make it KMS-backed.

### Why KMS-backed Vault?
When Vault is KMS-backed, it uses a cloud provider's Key Management Service to:
- Automatically unseal itself at startup (no need to enter Shamir key shares)
- Secure the Vault master key envelope (used to decrypt the data encryption key)
- Let you keep control of root encryption keys in your cloud account

Auto-unsealing Vault with a cloud KMS gives you:

🔐 Better operational security (no human key share handling)

🔁 Easier restarts/reboots/recovery

💥 Cleaner automation for production Vault clusters

### Why Azure Key Vault?
I'll use Azure Key Vault as my KMS only because I have some other very cheap, long-lived resources in the same Resource Group in Azure. I use a [delete lock](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources?tabs=json) to make sure these don't get cleaned up accidentally. 

### How-to

#### Azure set up

- Create a Key Vault in Azure.
- After creation, I used the IAM tab to give myself two roles. I believe I need the first in order to give myself the second, in order to create keys in the vault.
  - "Key Vault Data Access Administrator"
  - "Key Vault Administrator"
- Create a key in the vault called "vault-unseal-key"
- Create an App Registration called "oleary-kv-vault-auto-unseal"
  - In Azure Key Vault, give this app registration the role of "Key Vault Crypto User"

<figure>
    <a href="/assets/vault-setup-kms/key-vault-iam.png"><img src="/assets/vault-setup-kms/key-vault-iam.png"></a><caption>Screenshot of IAM permissions on Key Vault. Don't forget to also create the key.</caption>
</figure>

#### Vault set up

- Install Vault
- Create the following file at `/etc/vault.d/vault.hcl`
{% highlight hcl %}
storage "file" {
  path = "/opt/vault/data"
}

seal "azurekeyvault" {
  tenant_id      = "YOUR_TENANT_ID"
  client_id      = "YOUR_CLIENT_ID"
  client_secret  = "YOUR_CLIENT_SECRET"
  vault_name     = "YOUR_AZURE_KEY_VAULT_NAME"
  key_name       = "vault-unseal-key"
}

# HTTP listener
#listener "tcp" {
#  address = "127.0.0.1:8200"
#  tls_disable = 1
#}

# HTTPS listener
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/tls.crt"
  tls_key_file  = "/opt/vault/tls/tls.key"
}


ui = true
{% endhighlight %}

Now ensure that we can start and unseal Vault automatically. Create a systemd unit file at `/etc/systemd/system/vault.service`:

{% highlight conf%}
[Unit]
Description=Vault
Requires=network-online.target
After=network-online.target

[Service]
User=vault
Group=vault
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP \$MAINPID
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
{% endhighlight %}

Also make sure you create the Vault user and directory:
````bash
sudo useradd --system --home /etc/vault.d --shell /bin/false vault
sudo mkdir -p /opt/vault/data
sudo chown -R vault:vault /opt/vault /etc/vault.d
````
Now, start and initialize Vault:
````bash
sudo systemctl daemon-reexec
sudo systemctl enable vault
sudo systemctl start vault

export VAULT_ADDR='https://127.0.0.1:8200'
vault operator init -tls-skip-verify
````

### Initial configuration
For the sake of completeness, I'll make notes of my initial set up commands.

After the `vault operator init` command above, the CLI outputs a success message with an initial root token. I have used the vault CLI to login:
````bash
vault login <root token>
````
I also want to enable the AppRole auth method, and I have created an AppRole called **f5ast-role** so that I can access Vault from my AST container:
````bash
vault auth enable approle
vault write auth/approle/role/f5ast-role \
  token_ttl=60m \
  token_max_ttl=120m \
  secret_id_ttl=60m \
  policies="default"
````

#### Notes on HTTPS, UI, and preferences
- Notice I have the Vault server listening on HTTPS, with HTTP commented out. Because I'm using the default self-signed certs, I have also used `-tls-skip-verify` in my vault cli command, and/or I could set an environment variable with `export VAULT_SKIP_VERIFY=1`.
- Note, there is also a UI available at `https://[vault-server-ip]:8200`
- I can run `vault status` to verify that Vault is running.
- `vault -autocomplete-install` and reloading the bash shell allows CLI autocomplete.

### Conclusion
We now have Vault set up on Ubuntu, and on start up it will automatically unseal using the key from Azure Key Vault. We have initialized Vault, saved our root token, logged into Vault from the local CLI, enabled AppRole auth method and created an AppRole.