---
layout: single
title: "Migrating to Mac, part 1"
date: 2026-11-14
categories: [mac]
tags: [windows, mac]
excerpt: "I'm finally migrating from Windows to Mac for primary workstation. Here's why and how"
---

## Summary

I'm moving from a Win 11 laptop that [I set up 2 years ago]({% post_url 2024-06-15-new-laptop-launch-pagent-auto-load-key %}) to a MacBook. I still intend to run almost nothing on the laptop locally, but little things have been bugging me. Battery life, scripts that devs write that only work on Mac, and other little things like this [crazy work around]({% post_url 2026-07-26-alpine-jumphost %}) I do to get to my Linux VM when connected to VPN, or the fact that I had to [document my static IP]({% post_url 2025-03-26-HyperV-internal-virtual-switch-static-IP %}) method for my internal virtual switch.

I like to do any kind of dev work on a "remote" Ubuntu VM because my laptop stays clean and I must specify my dependencies. I want to take this a step further now that I'm starting with a clean slate: ideally *never* modify a running VM. Just delete it and deploy a new, updated VM.

On Windows my VM lived in Hyper-V and the rebuild was a [runbook I followed]({% post_url 2024-06-07-ubuntu-server-post-deploy-script %}). On the Mac it's Lima plus one YAML file in git.

This post covers why I picked Lima, what the config looks like, how networking differs from my Hyper-V setup, and how it compares with the earlier posts where I solved the same problems the hard way. 

## The old setup: Windows, Hyper-V and one disposable Ubuntu VM

It worked, but it was a manual attempt at best practices


| Pros | Cons |
| --- | --- |
| 80% followed best practices | Still some lingering things that were running on the Windows host |
| Windows is easy to manage | Windows felt like it was not as slick as Mac. I admit that's not a technical reason |
| Hyper-V is free | There's native virtualization on Mac so you don't need Parallels or Fusion. |
| Almost everything runs on Windows | Once in a while I find a tool that only runs on Mac/Linux |
| Windows can run WSL | But still, there's scripts from colleagues that assume a Mac |

## The new setup: Lima on the Mac's built-in virtualization

On the MacBook I run the same Ubuntu VM with [Lima](https://lima-vm.io). I expected to need Parallels, but macOS has its own virtualization APIs (Hypervisor.framework and Virtualization.framework), and Lima can use them directly. That's the Mac counterpart to Hyper-V: no third-party hypervisor required.

What made Lima the pick:

- **The VM is defined in a YAML file.** Image, CPU, memory, disk, mounts and provisioning scripts are all in one file I can keep in git.
- **Rebuilding is two commands.** `limactl delete` and `limactl start` replace my hand-followed checklist.
- **It's scriptable and free**, which fits the "nothing special on the host" goal.

Two choices I made on purpose:

- **A minimal Ubuntu cloud image** instead of the default server image, to keep the VM small.
- **No mount of my Mac home directory.** Code lives inside the VM, and files move by git or `limactl cp`. A shared mount would blur the line between host and VM that I'm trying to keep sharp.

The only things I installed so far on the Mac itself are Homebrew (to get Lima) and VS Code.

## The config and the rebuild loop

This is the whole definition of my VM, saved as `dev.yaml`:

```yaml
# dev.yaml — minimal Ubuntu 24.04 dev VM
vmType: vz            # Apple Virtualization.framework (macOS 13+)
cpus: 4
memory: "4GiB"
disk: "30GiB"

images:
  - location: "https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-arm64.img"
    arch: "aarch64"
  - location: "https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img"
    arch: "x86_64"

# Keep the host out of the VM: no Mac home mount
mounts: []

# No container runtimes unless you want them
containerd:
  system: false
  user: false

ssh:
  forwardAgent: true

provision:
  - mode: system
    script: |
      #!/bin/bash
      set -eux
      apt-get update
      apt-get install -y --no-install-recommends \
        git curl ca-certificates vim-tiny jq
  - mode: user
    script: |
      #!/bin/bash
      set -eux
      # clone your dotfiles here, e.g.:
      # git clone https://github.com/<you>/dotfiles ~/dotfiles && ~/dotfiles/install.sh
```

On Apple Silicon, Lima picks the `aarch64` image automatically, so the x86_64 entry is only there in case I share the file with an Intel machine.

Then:

```bash
limactl start --name=dev ./dev.yaml   # create and boot
limactl shell dev                     # get a shell
limactl delete dev                    # blow it away
```

Compare the provisioning block to my old post-deploy script: the same kind of steps (packages, tools, dotfiles), but run by the VM on first boot instead of typed by me.

## Installing Jekyll to preview this post

My old Ubuntu VM with Jekyll is gone, so I have to reinstall it before I can preview a post. I'm doing it inside the Lima VM rather than on the Mac, which keeps the "nothing on the host" rule intact. These are the steps from Jekyll's Ubuntu instructions, with gems installed in my home directory instead of system-wide:

```bash
sudo apt update
sudo apt install -y ruby-full build-essential zlib1g-dev

echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

gem install jekyll bundler
```

Then, from the directory of my blog repo cloned inside the VM:

```bash
bundle install
bundle exec jekyll serve
```

Lima forwards listening ports to `localhost`, so I open `http://localhost:4000` in the Mac's browser.

This is one-off setup typed by hand, which is exactly what I'm trying to stop doing. Installing Jekyll belongs in the `provision` block of `dev.yaml`, and I'll move it there when I get a chance.

## Networking, VPN and VS Code

By default, Lima's VM is **not** on a subnet my Mac can route to. It uses user-mode networking, so the VM gets a private NAT address (typically `192.168.5.x`) that the Mac can't reach directly. Three things make that fine in practice:

- Lima forwards ports the VM listens on to `localhost` on the Mac.
- SSH goes through a forwarded localhost port, and `limactl shell` handles that for me.
- The VM reaches the internet through NAT and reaches the Mac at `host.lima.internal`.

That's easier than Hyper-V, where I had to build an Internal switch, pick a subnet, set static IPs and add a NAT before the VM was usable they way I wanted.

So for VS Code Remote SSH, which I use on VPN all the time, I'm using the localhost forward instead of the VM's IP. Lima writes an SSH config for each instance. I've added `Include ~/.lima/dev/ssh.config` to `~/.ssh/config` which makes `lima-dev` host appear in VS Code as a remote SSH host.

The F5 Access VPN client blocked my laptop from the Hyper-V Internal switch subnet, which is why I had the crazy workaround I mentioned earlier. This local port forwarding means I don't have to choose between being on VPN and reaching my Ubuntu VM over SSH.

## How this compares with my Hyper-V posts

The goal hasn't changed since those posts: a disposable VM and a clean laptop I could swap at any time. It's easier to work with, faster, and more secure. I think this is a step forward.

| | Hyper-V on Windows (earlier posts) | Lima on the Mac |
| --- | --- | --- |
| Host tools | VS Code, Hyper-V, WinSCP, PuTTY and Pageant | Homebrew (for Lima), VS Code |
| Hypervisor | Hyper-V | macOS Virtualization.framework, through Lima |
| VM definition | Quick Create image plus a post-deploy checklist | `dev.yaml`, provisioned on first boot |
| Rebuild | Follow the runbook; resize the disk and paste in a key by hand | `limactl delete`, then `limactl start` |
| Default networking | Custom Internal switch, static IPs, NAT on 172.20.100.0/24 | User-mode NAT with forwarded localhost ports; nothing to configure |
| Stable address | Static IP on both the host adapter and the guest | Localhost forward; I don't care what the IP of the VM is |
| VPN | F5 VPN blocked the Internal switch; Alpine jump host with `ProxyJump` | Should be fine, have not tested yet|

The jump host series [part 1]({% post_url 2026-07-26-alpine-jumphost %}) and [part 2]({% post_url 2026-07-26-alpine-jumphost %}) is what I most want to avoid repeating.

## Caveats and what's next

- **The minimal image is minimal.** It ships with fewer packages than the standard server image, so expect to add a few things you took for granted.
- **Editing a live VM is an anti-pattern, and I did it anyway.** Lima copies the config into the instance at creation, so editing `dev.yaml` doesn't change a running VM; `limactl edit dev` does. Provisioning scripts only run on first boot, so some changes won't apply to an existing disk. The honest fix is to make the same change in `dev.yaml` and rebuild.

**Sources**

- [Lima](https://lima-vm.io)
- [Lima setup blog post](https://solanyum.com/en/blog/macos-lima-ile-izole-ubuntu-sanal-makine-kurulumu)