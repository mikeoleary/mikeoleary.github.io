---
layout: single
title: "Migrating to Mac, part 2"
date: 2026-11-14
categories: [mac]
tags: [windows, mac]
excerpt: "Quick OS distro update"
---

## Summary

Last post explained why I'm moving to Mac and my new Ubuntu VM setup. After getting annoyed with a slim OS I want to use a more standard Ubuntu image.

## The change to my YAML file

Old:

```yaml
images:
  - location: "https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-arm64.img"
    arch: "aarch64"
  - location: "https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img"
    arch: "x86_64"
```

### New:

```yaml
images:
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-arm64.img"
    arch: "aarch64"
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img"
    arch: "x86_64"
```

The difference is the size of the image files. The old one was about 213 MB, and the new one is about 590 MB. I had originally opted for a smaller image because I was on airplane WiFi and didn't want to pull down a huge image.

The repo is [here](https://github.com/mikeoleary/lima-vm-configs) if you want to see the current dev.yaml file.