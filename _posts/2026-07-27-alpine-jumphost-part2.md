---
title: "Alpine jump host, part 2: WinSCP, outbound internet, and forwarding other ports"
excerpt: "A few follow-up fixes after putting my Hyper-V jump host into daily use: WinSCP needs its own tunnel config, outbound internet from the internal VM needed a kernel setting, and forwarding non-SSH traffic needed real DNAT rules."
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
    <a href="/assets/alpine-jumphost/network-routing2.jpg"><img src="/assets/alpine-jumphost/network-routing2.jpg"></a>
    <figcaption>Updates on workarounds that multiple NIC's allow</figcaption>
</figure>

### Summary

After [getting the Alpine jump host working]({{ site.url }}/hyper-v/reaching-hyperv-internal-network-on-vpn/) for SSH access to my internal Hyper-V VM while on VPN, a few real-world gaps showed up once I started actually using it day to day. This post covers three follow-up issues and how I fixed each one.

### 1. WinSCP doesn't read my SSH config file

VSCode's Remote-SSH extension shells out to the real `ssh.exe`, so it picked up my `~/.ssh/config` `ProxyJump` entry automatically, with zero extra config. WinSCP does not — it has its own SSH implementation and never reads that file, so `ProxyJump` is invisible to it.

The fix is to configure the jump manually as a **Tunnel** inside WinSCP itself:

1. Session settings → **Advanced** → **Connection → Tunnel**
2. Check **Connect through SSH tunnel**
3. Tunnel host name: the jump host's external IP
4. Tunnel user name + private key (converted to `.ppk` via WinSCP's bundled PuTTYgen if needed)
5. Leave the main session's host name as the target VM's internal IP

Once that's set, WinSCP opens the tunnel to the jump host first, then connects on to the target VM through it — same end result as `ProxyJump`, just configured in WinSCP's own UI instead of inherited from a shared config file.

### 2. Outbound internet from the internal VM stopped working

Separately from the SSH/inbound story, I noticed my internal Ubuntu VM couldn't reach the internet at all while on VPN (`apt-get update` and friends). This is a different traffic direction than the jump host was originally built for, so it needed its own fix.

I pointed the Ubuntu VM's default gateway at the Alpine jump host's internal IP, and added `FORWARD` rules plus `MASQUERADE` for outbound traffic — the same shape of rule as the SSH forwarding, just for return-path traffic heading out to the internet instead of into the internal switch.

The rules alone didn't get it working immediately, though. Troubleshooting turned up two separate issues:

- **Strict reverse-path filtering (`rp_filter`).** With `rp_filter=1` (strict) on both interfaces, the kernel was silently dropping forwarded packets before they reached `eth0`, even though the routes and FORWARD rules were correct. Disabling it (`rp_filter=0` on both interfaces, plus `all`/`default`) cleared that up:
  ```sh
  echo 0 > /proc/sys/net/ipv4/conf/eth0/rp_filter
  echo 0 > /proc/sys/net/ipv4/conf/eth1/rp_filter
  echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
  echo 0 > /proc/sys/net/ipv4/conf/default/rp_filter
  ```
- **`net.ipv4.ip_forward` was actually still `0`.** This was the real root cause — my earlier `sysctl -p` hadn't actually applied it (or something reset it), and I didn't catch it until I checked `sysctl net.ipv4.ip_forward` directly rather than assuming the config file entry meant it was live. With forwarding disabled at the kernel level, packets arriving on the internal NIC were dropped before ever reaching the FORWARD chain — which is why the FORWARD counters stayed at zero no matter what rules I added.

Once `ip_forward` was actually confirmed as `1`, outbound internet access from the internal VM started working immediately, routed through Alpine's own MASQUERADE'd IP on the way out.

**Lesson:** when packets aren't reaching a chain at all (0 packets/0 bytes on every rule), check the more fundamental kernel switches first (`ip_forward`, `rp_filter`) before troubleshooting the iptables rules themselves — a correct ruleset won't matter if forwarding is off at the kernel level.

### 3. Forwarding other ports besides SSH

The original setup only handled SSH (port 22) via `ProxyJump`/tunneling, which works because it's a client-initiated hop through an existing SSH session. But I also wanted to reach a plain HTTP service running on the internal VM (port 4000) directly by hitting the jump host's external IP — no SSH involved at all.

That needs actual **DNAT (destination NAT / port forwarding)**, not just the FORWARD-and-MASQUERADE setup used for outbound traffic:

```sh
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 4000 -j DNAT --to-destination 172.20.100.101:4000
iptables -A FORWARD -i eth0 -o eth1 -p tcp -d 172.20.100.101 --dport 4000 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

The second rule mattered as much as the first: my existing `eth0 → eth1` FORWARD rule only permitted `RELATED,ESTABLISHED` traffic (since it was written for SSH *return* traffic, not fresh inbound connections). A brand-new inbound connection to port 4000 doesn't match either of those states, so without explicitly allowing `NEW` traffic for that specific port, the DNAT rule alone wasn't enough — the first packet still got dropped at FORWARD.

With both rules in place, `http://<jump-host-external-ip>:4000` now reaches the internal VM's service directly.

### Where things stand now

- SSH/SFTP via CLI and VSCode: works transparently through `~/.ssh/config`, on or off VPN.
- WinSCP: works, via a manually configured tunnel (not automatic).
- Outbound internet from the internal VM: works, routed through Alpine.
- Other TCP services on the internal VM: reachable via explicit DNAT rules per port, as needed.

Bit by bit this jump host has turned into the actual gateway for my internal switch, in both directions — worth keeping this post as the reference for which rule handles which direction next time I need to add another port.