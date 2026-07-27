---
title: "Reaching my Hyper-V internal network while connected to VPN"
excerpt: "My corporate VPN client blocks access to my Hyper-V internal switch. Here's how I worked around it with a tiny Alpine jump host."
categories:
  - hyper-v
tags:
  - hyper-v
  - networking
  - vpn
  - alpine
toc: true
toc_label: "On this page"
---

<figure>
    <a href="/assets/alpine-jumphost/network-routing.jpg"><img src="/assets/alpine-jumphost/network-routing.jpg"></a>
    <figcaption>Multiple NICs allow for many things</figcaption>
</figure>


### Summary

I run my daily-use Ubuntu VM on a custom Hyper-V Internal virtual switch (see my [previous post]({{ site.url }}/hyper-v/HyperV-internal-virtual-switch-static-IP/) on giving it a persistent static IP). That setup works great — until I connect to my corporate VPN (F5 Access / BIG-IP APM), at which point the VM becomes completely unreachable from my laptop. The fix wasn't a routing tweak, it was a small Alpine Linux VM acting as a jump host with one leg on my physical LAN and one leg on the internal switch.

### The problem

Connecting to VPN broke SSH/WinSCP/VSCode access to my internal VMs. My first assumption was routing: maybe the VPN client was pushing a full-tunnel default route that was winning over my internal subnet.

Turned out to be wrong on both counts, but it took some diagnosis to get there.

#### Checking the routing table

Before/after connecting to VPN, I compared `route print` output. The route to my internal switch's subnet (`172.20.100.0/24`) was actually still present and correctly installed as a persistent route:

```
Network Destination        Netmask          Gateway       Interface  Metric
172.20.100.0     255.255.255.0   On-link      172.20.100.1     16
```

Since this is a more specific prefix than the VPN's `0.0.0.0/0` default route, longest-prefix-match rules mean it should always win regardless of metric. So the routing table wasn't lying to me — the route was there, on-link, correct.

And yet, ping to my VM and even to my own internal switch's gateway (`172.20.100.1`) went completely silent while on VPN. No "destination unreachable," just nothing. That silence is the tell: a route being present but traffic still not passing means something below the routing layer is filtering it.

#### It's not routing — it's the VPN client's packet filter

F5 Access (like most enterprise VPN clients doing split-tunnel enforcement) installs a **Windows Filtering Platform (WFP)** callout driver that blocks traffic to non-tunnel destinations at the kernel level, independent of the routing table. This is deliberate anti-split-tunnel policy enforcement, controlled server-side by the BIG-IP APM Access Policy (a "Local Subnet Access" setting, or a split-tunnel exclude list) — not something fixable from the client side or with local admin rights.

I confirmed the scope of the block with a couple of pings while connected to VPN:

- `ping 172.20.100.1` (my own internal switch gateway) — **failed**
- `ping 192.168.1.1` (my physical LAN gateway) — **succeeded**

So the block is scoped specifically to the Hyper-V internal-switch adapter, not a blanket "nothing but the tunnel" policy. That distinction is what made a workaround possible at all.

### What I considered that didn't work

For completeness, here's what I ruled out before landing on the jump host:

- **Bumping the interface metric** on the internal vEthernet adapter — didn't matter, since the route was never the problem.
- **Adding a persistent static route** with admin rights (`route -p add ...`) — the route survived the VPN connection just fine, but traffic still didn't pass. This is what confirmed it was a filter, not a route.
- **Trying to remove/override the WFP filter directly** — technically possible with admin rights, but a bad idea in practice: the F5 client service re-asserts its policy on a timer or reconnect, so any local override would likely get stomped anyway. More importantly, this is my own company's security policy working as designed so I didn't touch. Also, I'm out of my depth here, guided with AI tools but am not confident here anyway.

The actual fix for the root cause is a server-side APM policy change (Local Subnet Access, or excluding `172.20.100.0/24` from the tunnel) — that's a conversation with whoever owns the BIG-IP policy, not something to solve locally. In the meantime, I wanted a workaround.

### The workaround: a jump host VM

The idea: a small VM with one NIC on my physical LAN (External switch) and one NIC on my internal switch, sitting between my laptop and my other VMs. Since traffic to my physical LAN gateway survives the VPN's filter, SSH-ing to the jump host's LAN-facing IP works fine even on VPN — and from there, it's a local hop to the internal switch, which the VPN never touches.

```
Host (VPN on) → SSH to jump host on 192.168.1.x → forwards to target VM on 172.20.100.x
```

#### VM vs. container

I considered a Windows container with the `transparent` network driver instead of a full VM, mainly to save on footprint. Decided against it:

- Docker Desktop's default WSL2 backend shares one NAT'd network view across containers — no clean way to attach a container directly to two different Hyper-V virtual switches.
- The `transparent` driver *can* attach a container to a vSwitch directly, but that requires Windows Server Core/Nano containers, not Linux containers, and is a much less common, less documented setup.
- A bare Alpine VM ends up with a smaller footprint anyway than Docker Desktop's backing WSL2 VM, so the "container to save memory" reasoning didn't actually hold up once I compared the numbers.

A VM with two vNICs attached to two switches is just how Hyper-V networking normally works — nothing exotic, nothing to fight.

#### Building the jump host

- **OS**: Alpine Linux, `alpine-virt` edition (the ISO built for VM use) — smallest realistic option with normal package management, `iptables`, and OpenSSH support.
- **VM generation**: started on Generation 2, but hit a boot failure — the VM would fall through to PXE boot even with Hard Drive first in the firmware boot order and Secure Boot disabled. Switched to **Generation 1** (legacy BIOS/MBR boot) and the install was reliable. For a small headless VM with no need for Gen2-specific features, Gen1 is the simpler choice anyway.
- **NICs**: one attached to my External virtual switch (physical LAN), one attached to my existing Internal virtual switch (`172.20.100.0/24`).
- **IP forwarding + NAT**:
  ```sh
  echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
  sysctl -p
  apk add iptables iptables-openrc
  iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
  rc-update add iptables
  /etc/init.d/iptables save
  ```
- **SSH**: created a non-root user, added my public key to `~/.ssh/authorized_keys`, enabled and started `sshd`. Had to explicitly set `AllowTcpForwarding yes` in `sshd_config` — it was blocking the `ProxyJump` hop with an "administratively prohibited" error until I turned it on.

#### Making it transparent from Windows

A couple of entries in `~/.ssh/config` on my laptop:

```
Host jumphost
    HostName 192.168.1.x
    User jumpuser

Host devvm
    HostName 172.20.100.101
    User <my-vm-user>
    ProxyJump jumphost
```

With that in place, `ssh devvm` (or `ssh 172.20.100.101` directly, since the config matches on hostname) transparently hops through the jump host. No change needed in WinSCP or VSCode Remote-SSH either, since both understand `ProxyJump`.

I don't actually work from a PowerShell or CMD prompt day to day — I live in VSCode. Since VSCode's Remote-SSH extension reads the exact same `~/.ssh/config` file the `ssh` CLI does, this fix applied with zero changes on my end there. I opened VSCode on VPN, connected to the same host entry I always use, and it just worked — routed through the jump host without me touching any settings.

### The final result

- Off VPN: my Ubuntu dev VM is reachable directly on `172.20.100.101`, same as always.
- On VPN: SSH to `172.20.100.101` is automatically routed via the jump host at `192.168.1.205`, and it works — no change to my daily workflow, no manual jump-host step to remember.

### Feedback and summary

The lesson that took the longest to land: a correct, present route in `route print` doesn't mean traffic will actually pass — WFP-level filtering happens below routing and won't show up there at all. Ping tests to a couple of different gateways (internal switch vs. physical LAN) were what actually isolated the scope of the block and pointed me at a workaround instead of a dead end.

The proper long-term fix is still a policy change on the APM side, and I'll follow up on that separately. This jump host is a workaround, not a fix — but it's a workaround that survives reboots, VPN reconnects, and switching between Ethernet and WiFi, so it's good enough for now.