---
layout: single
title:  "My Ubuntu server post-deploy script"
categories: [linux]
tags: [linux]
excerpt: "This is intended to be updated. This is my post deploy steps when using Hyper-V and Quick Create for Ubuntu 22.04" #this is a custom variable meant for a short description to be displayed on home page
---
#### Background
For most daily Linux use, I run Ubuntu 22.04 as a VM in Hyper-V on my Windows 10 laptop. I am sure others prefer a different way, but I like this setup. Win 10 is easiest to use in the corporate environment, and I have a Linux VM that feels local. It's a server OS, and I can blow it away at any time. 

Occasionally I will accidentally hose the machine and need to start from scratch. Inevitably I forget what and how I've installed packages, so I'm finally going to document a quick post-deploy script.
```bash

## First, Quick Create with Hyper-V often creates only a 12 GB disk. 
# Shut down VM
# Use Hyper-V to extend disk to 128 GB. Instructions: https://linguist.is/2020/08/12/expand-ubuntu-disk-after-hyper-v-quick-create/
# Start up VM
sudo apt install cloud-guest-utils
sudo fdisk -l # this should show a DISK called /dev/sda and a DEVICE called /dev/sda1 with a size of 128 GB now.
sudo df -l # this should show a filesystem called /dev/sda1 but it won't be using all of the 128 GB yet
# Expand the sda1 partition into the free space
sudo growpart /dev/sda 1
# run resize2fs
sudo resize2fs /dev/sda1
sudo df -l ## this should now show the same filesystem, /dev/sda1, but now there is much more free space left.

## INSTALL openssh-server

sudo apt install openssh-server -y

## create authorized keys file

mkdir ~/.ssh
cd ~/.ssh
touch authorized_keys
# (vi and paste in public key, save and exit)

## Install curl
sudo apt-get update
sudo apt-get install curl -y

## Install AZ CLI
# full instructions from Microsoft: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

## Install AWS CLI
# full instructions from AWS: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

## Install kubectl 
# instructions from k8s docs here: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# create .kube directory in user's home directory and also enable kubectl autocompletion for the user
mkdir ~/.kube
echo 'source <(kubectl completion bash)' >>~/.bashrc

## Install Git
sudo apt install git-all -y

## Install Jekyll so I can work on my blog
# https://jekyllrb.com/docs/installation/ubuntu/
sudo apt-get install ruby-full build-essential zlib1g-dev -y

echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

gem install jekyll bundler

## Optional
#sudo apt install python3-pip -y

```

 I'm often told to use WSL2, but I like having a VM so that I can blow away/ roll back/ etc. In any case, I'll update this script if/when I need.