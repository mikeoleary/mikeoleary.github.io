---
layout: single
title:  "Using SOPS locally on Docker host for secret injection"
categories: [docker]
tags: [docker]
excerpt: "Here's an imperfect but workable solution for secrets management using SOPS and Docker Compose" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/sops-docker-compose-2/containers_and_secrets.webp"><img src="/assets/sops-docker-compose-2/containers_and_secrets.webp"></a>
</figure>

### Summary
One option for supplying secrets to Docker Compose is to use a tool like SOPS locally on the Docker host to decrypt secrets, and then supply those values to Docker Compose as securely as possible.

### Supporting articles
This article was inspired by a [previous post]({% post_url 2025-02-28-sops-docker-compose %}) in which I talked about the sidecar method of providing a secret to a container. The sidecar container has a secret and provides it, via a shared volume mount or shared container network, to the app container.

I also critiqued an article which I'd used to learn some secret management options. 

Here's an update: I reached out to GitGuardian and they updated their article. They agreed the original article was likely incorrect (but the author has moved on so couldn't be reached). They fixed a couple typos and removed example #4. Thanks, good folks at GitGuardian!

### Another option: use SOPS locally on the Docker host
When discussing with GitGuardian, it occured to me that example #4 of the article (at least, the version of the article at that time), was probably meant to refer to running SOPS *locally*, and *not* as a sidecar container. That would mean the Docker Compose sample file should not have included a SOPS container. 

I believe an example of running SOPS locally and providing the secrets as environment variables from the host was what the author intended. Here's an example of how to do that.

#### Install Docker and SOPS locally
I have an Ubuntu 22.04 host. Let's install Docker, Docker Compose, and [install SOPS](https://github.com/getsops/sops/releases). For now, I'm going to install Docker the way I [typically do it]({% post_url 2023-10-12-docker-install-run-helloworld %}) but I do plan to change the way I install Docker in the near future. 

To that end, I'll also install the Docker Compose CLI plugin manually following [these steps](https://docs.docker.com/compose/install/linux/#install-the-plugin-manually).

```bash
sudo apt update
sudo apt install docker.io -y

# start and enable service
sudo systemctl start docker
sudo systemctl enable docker

# to check and see if Ubuntu is running the docker service, uncomment below.
#systemctl status docker
 
# Add your user to Docker group to avoid typing sudo everytime you run docker commands.
sudo usermod -aG docker ${USER}
newgrp docker

# Manually install Docker Compose v 2.34.0, x86_64, for the local user
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.34.0/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose

chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose

# Install SOPS
# Download the binary
curl -LO https://github.com/getsops/sops/releases/download/v3.9.4/sops-v3.9.4.linux.amd64

# Move the binary in to your PATH
mv sops-v3.9.4.linux.amd64 /usr/local/bin/sops

# Make the binary executable
chmod +x /usr/local/bin/sops
```

#### Generate a PGP encryption key
SOPS works with keys held in various systems, but for the purpose of this article we'll generate and use a PGP key. BTW, I originally found [this doc](https://gist.github.com/twolfson/01d515258eef8bdbda4f) a nice primer on learning gpg in a hurry, and [this doc](https://blog.gitguardian.com/a-comprehensive-guide-to-sops/) for clearing up PGP vs GPG.

`gpg` has come pre-installed on my Ubuntu VM so install it if you have to, and then let's generate a key:

```bash
export KEY_NAME="Michael O"
export KEY_COMMENT="test key for sops"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: ${KEY_COMMENT}
Name-Real: ${KEY_NAME}
EOF
```

Now let's get the fingerprint of the key we just created. Note my output below with the key fingerprint of `1216AFF741C3AE5D9F4F0F256BA169C2973F8317`

{% highlight bash %}
$ gpg --list-secret-keys --keyid-format LONG $KEY_NAME
sec   rsa4096/6BA169C2973F8317 2025-03-27 [SCEAR]
      1216AFF741C3AE5D9F4F0F256BA169C2973F8317
uid                 [ultimate] Michael O (test key for sops)
ssb   rsa4096/AAB98F99864AFE63 2025-03-27 [SEA]
{% endhighlight %}

#### Encrypt a password
With our new key, let's use SOPS to encrypt a password of `ilovechocolate`.

```bash
echo "ilovechocolate" | sops encrypt --pgp 1216AFF741C3AE5D9F4F0F256BA169C2973F8317 /dev/stdin > secret.enc.txt

sops decrypt secret.enc.txt
```

Your output from the second command should show the decrypted text.

#### Encrypt password, decrypt to environment variable
Now that we know how SOPS works, let's encrypt something that we intend to decrypt and load as an environment variable. The `exec-env` function of sops works with structured files (YAML or JSON) to extract the values as **temporary environment variables** that are passed to a child process.

```bash
# create a JSON-formatted file containing our password
cat > password.json << EOF
{
  "BIGIP_PASSWORD_1":"SuperSecretPassword"
}
EOF

# encrypt our JSON-formatted file and remove the decrypted file.
sops encrypt --pgp 1216AFF741C3AE5D9F4F0F256BA169C2973F8317 password.json > password.enc.json && rm password.json

# decrypt our file with exec-env, which will populate temporary env vars for the child process
sops exec-env password.enc.json 'echo $BIGIP_PASSWORD_1'
```

#### (Insecure) Decrypt our password and save a local environment variable
Now we have an encrypted file that we can decrypt at any time and have the value populated as an environment variable for a child process. It is possible to use `exec-env` to save environment variables on the Docker host by having the child process create them, like this:

```bash
# decrypt our file with exec-env but have the child process create an env var in the open shell
eval "$(sops exec-env password.enc.json 'echo BIGIP_PASSWORD_1=$BIGIP_PASSWORD_1')"
```

Then we could use this environment variable easily. However, there's a more secure way.

#### (Secure) Decrypt our password and pass the value to Docker Compose
Let's use the child process to run Docker Compose.

In the same directory in which the `password.enc.json` file exists, create `docker-compose.yaml`:

```yaml
services:
  env_printer:
    image: alpine
    command: [ "sh", "-c", "echo The secret is: $BIGIP_PASSWORD_1" ]
    environment:
      - BIGIP_PASSWORD_1

```

Now, run SOPS to decrypt your secret, populate an environment variable for the container, and have the container run and print this value.

```bash
# decrypt our file with exec-env and run docker compose which can use these temporary env vars.
sops exec-env password.enc.json 'docker compose up'
```

Notice that this environment variable is **not** present on the Docker host shell in which you are typing commands, but only the child process that ran Docker Compose.

### Conclusion
You can run SOPS as a tool to decrypt secrets and provide them as **temporary** environment variables to Docker Compose.