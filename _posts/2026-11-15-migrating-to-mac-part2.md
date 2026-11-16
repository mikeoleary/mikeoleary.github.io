---
layout: single
title: "Migrating to Mac, part 2"
date: 2026-11-15
categories: [mac]
tags: [windows, mac]
excerpt: "Quick OS distro update, ca certs, and blog repo set up"
---

## Summary

Last post explained why I'm moving to Mac and my new Ubuntu VM setup. After getting annoyed with a slim OS I want to use a more standard Ubuntu image, I needed ca certs for my enterprise TLS decryption tools, and I want to automatically clone my blog repo and install jekyll and dependencies so that I can work on my blog without having to remember how to set it all up. 

## The updated YAML file:

I'm basically doing this now:
- updated OS image
- installing CA Certs I want (in order to be able to clone a repo from github, since github traffic is inspected by my enterprise tooling)
- cloning a repo so it's always there when my VM boots
- installing jekyll and dependencies so I can work on my blog right away from my newly-launched VM

```yaml
# dev.yaml — minimal Ubuntu 24.04 dev VM
vmType: vz            # Apple Virtualization.framework (macOS 13+)
cpus: 4
memory: "4GiB"
disk: "30GiB"


images:
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-arm64.img"
    arch: "aarch64"
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img"
    arch: "x86_64"

# No container runtimes unless you want them
containerd:
  system: false
  user: false

ssh:
  forwardAgent: true

mounts:
  - location: "~/certs"      # Path on Mac containing netskope-ca.crt
    mountPoint: "/mnt/certs" # Explicit path inside the Lima VM
    writable: false

provision:
  - mode: system
    script: |
      #!/bin/bash
      set -eux
      
      # 1. Install corporate CA certificates if present
      if [ -d /mnt/certs ]; then
        cp /mnt/certs/_.dfw3.goskope.com.pem /usr/local/share/ca-certificates/_.dfw3.goskope.com.crt
        cp /mnt/certs/ca.f5.goskope.com.pem /usr/local/share/ca-certificates/ca.f5.goskope.com.crt
        update-ca-certificates
      fi
      
      # 2. Install base system dependencies and Ruby toolchain as root
      export DEBIAN_FRONTEND=noninteractive
      apt-get update
      apt-get install -y --no-install-recommends \
        git curl ca-certificates vim-tiny jq ruby-full build-essential zlib1g-dev

      # 3. Install Jekyll and Bundler globally so 'bundle' is available to all users
      gem install jekyll bundler


  - mode: user
    script: |
      #!/bin/bash
      set -eux
      
      # 4. Clone your GitHub repository into the user home directory
      if [ ! -d ~/mikeoleary.github.io ]; then
        git clone https://github.com/mikeoleary/mikeoleary.github.io ~/mikeoleary.github.io
      fi
      
      cd ~/mikeoleary.github.io
      
      # 5. Create a Gemfile if the repository doesn't have one yet
      if [ ! -f Gemfile ]; then
        cat << 'EOF' > Gemfile
      source "https://rubygems.org"
      gem "github-pages", group: :jekyll_plugins
      EOF
      fi
      
      # 6. Configure Bundler to install gems locally inside the repo (bypasses permission errors)
      bundle config set --local path 'vendor/bundle'
      
      # 7. Install all dependencies automatically on VM creation
      bundle install

```


The repo is [here](https://github.com/mikeoleary/lima-vm-configs) if you want to see the current dev.yaml file. (I'll probably keep the repo private for now)