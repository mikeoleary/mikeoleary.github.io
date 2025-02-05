---
layout: single
title:  "DHCP vs Static Routes + BIG-IP upgrade, uh oh!"
categories: [f5]
tags: [f5, big-ip]
excerpt: "In a recent customer case I had to find why the mgmt IP reset to 127.0.0.1. This is what I learned" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

<figure>
    <a href="/assets/dhcp-static-routes/conflicting-routes.webp"><img src="/assets/dhcp-static-routes/conflicting-routes.webp"></a>
</figure>

### Summary
I recently learned that the Azure DHCP server defines routes for DHCP clients, and that if you have a statically configured route that overlaps a DHCP-configured route, plus upgrade your BIG-IP, you may hit a problem.

### Details
Recently I had a customer reach out with basically this question:
> We upgraded our BIG-IP devices in Azure and lost connectivity to the management interface. When I deployed a jump host in the same subnet as mgmt interface, I was able to establish a connection via SSH and GUI. However the command `tmsh show sys ip-address all-properties` showed that the management IP address had reset to 127.0.0.1. Did I hit the issue described in [K52454288](https://my.f5.com/manage/s/article/K52454288)?

The answer was "yes" but the customer wanted more information, so I went about trying to replicate this strange issue where a device reset it's own mgmt IP address to 127.0.0.1

#### My test environment
I deployed a pair of BIG-IP devices with the following:

- the mgmt interface was set to DHCP in the BIG-IP GUI (this is a bad practice but it is what this customer accidentally had configured)
- The F5 CFE was installed. A management route to the Azure Metadata server was configured, as is required by [this step](https://clouddocs.f5.com/products/extensions/f5-cloud-failover/latest/userguide/azure.html#set-up-access-to-azure-s-instance-metadata-service) in the CFE installation instructions:

```bash
tmsh modify sys db config.allow.rfc3927 value enable
tmsh create sys management-route metadata-route network 169.254.169.254/32 gateway 10.0.0.1
tmsh save sys config
```

#### What is the problem?
Let's check out the file at `/var/lib/dhclient/dhclient.leases`. This shows the lease obtained by the DHCP client.

```
>[root@bigip2:Active:In Sync] config # more /var/lib/dhclient/dhclient.leases
lease {
  interface "mgmt";
  fixed-address 10.0.0.12;
  server-name "LVL021061819034";
  option subnet-mask 255.255.255.0;
  option routers 10.0.0.1;
  option dhcp-lease-time 4294967295;
  option dhcp-message-type 5;
  option domain-name-servers 168.63.129.16;
  option dhcp-server-identifier 168.63.129.16;
  option dhcp-renewal-time 4294967295;
  option dhcp-rebinding-time 4294967295;
  option unknown-245 a8:3f:81:10;
  option classless-static-routes 0 10.0.0.1,32.168.63.129.16 10.0.0.1,32.169.254.169.254 10.0.0.1;
  option domain-name "amfmy4g4wikuvfmrhabdvggqph.cx.internal.cloudapp.net";
  renew 0 2161/03/15 05:49:20;
  rebind 0 2161/03/15 05:49:20;
  expire 0 2161/03/15 05:49:20;
}
lease {
  interface "mgmt";
  fixed-address 10.0.0.12;
  server-name "LVL021061819034";
  option subnet-mask 255.255.255.0;
  option dhcp-lease-time 4294967295;
  option routers 10.0.0.1;
  option dhcp-message-type 5;
  option dhcp-server-identifier 168.63.129.16;
  option domain-name-servers 168.63.129.16;
  option dhcp-renewal-time 4294967295;
  option classless-static-routes 0 10.0.0.1,32.168.63.129.16 10.0.0.1,32.169.254.169.254 10.0.0.1;
  option unknown-245 a8:3f:81:10;
  option dhcp-rebinding-time 4294967295;
  option domain-name "amfmy4g4wikuvfmrhabdvggqph.cx.internal.cloudapp.net";
  renew 6 2161/03/14 22:37:07;
  rebind 6 2161/03/14 22:37:07;
  expire 6 2161/03/14 22:37:07;
}
```

I'm no expert, but it looks like DHCP created a route for 169.254.169.254/32 with next hop 10.0.0.1. This overlaps with my statically configured route.

#### Let's verify the problem
Use TMSH to look at the routes on the mgmt interface. Notice lines 13 through 21 show overlapping routes. One was created manually and one was created via DHCP. 

```
>tmsh list sys management-route

sys management-route default {
    description configured-statically
    gateway 10.0.0.1
    network default
}
sys management-route dhclient_route1 {
    description configured-by-dhcp
    gateway 10.0.0.1
    network 168.63.129.16/32
}
sys management-route dhclient_route2 {
    description configured-by-dhcp
    gateway 10.0.0.1
    network 169.254.169.254/32
}
sys management-route metadata-route {
    gateway 10.0.0.1
    network 169.254.159.254/32
}
```

Why is this even a problem? The routes *overlap* but they don't *conflict*. The gateway is 10.0.0.1 in either case.

I don't strictly know the answer. In fact, in my experience, if you now *reboot* this device, it comes back up without problems. *Upgrades* however seem to uncover a problem. Perhaps that's just my limited testing, but let's continue.

#### What happens when you upgrade?
Here's where my testing has not been 100% consistent. If the following things are all true, I seem to be able to replicate the problem:

- DHCP is enabled on mgmt interface
- a static mgmt route has been created for the 169.254.169.254/32 route
- The device boots into a new boot location
  - whether upgrading to newly-installed image, or simply reverting back to a previously-used boot location.

When I have tested this, I have replicated the issue about 50% of the time. I've only tested 5-6 times. I am likely missing more details. On the occasions I have replicated this issue, this is what I see when I run `tmsh show sys ip-address all-properties | grep management-ip`:

```bash
[root@bigip2:Active:In Sync] config # tmsh show sys ip-address all-properties | grep management-ip
10.0.0.11       cm device             bigip1.local     management-ip
127.0.0.1       cm device             bigip2.local     management-ip
```

That 127.0.0.1 is the problem. 

Let's check out `/var/log/boot.log` and notice that the dhcp client fails:
```
[root@bigip2:Active:In Sync] config # more /var/log/boot.log
.....
2025-02-05T08:07:09-08:00 bigip2.local notice boot_marker : ---===[ HD1.1 - BIG-IP 17.1.1.3 Build 0.0.5 ]===---
Feb  5 07:45:46 bigip2.local notice vadc-init[19940]: vadc-init: Run all the deployment-specific startup scripts first
Feb  5 07:45:46 bigip2.local notice vadc-init[19942]: vadc-init: Initializing cloud-init
Feb  5 07:45:46 bigip2.local notice vadc-init[19954]: vadc-init: vadc-init - DONE
Feb  5 08:03:29 bigip2.local err dhcp_config[31133]: save error: 01070734:3: Configuration error: invalid Management Route, the dest/netmask pair 169
.254.169.254/255.255.255.255 already exists for /Common/azureMetadata
Feb  5 08:03:29 bigip2.local err NET[31182]: dhclient: dhcp-config failed with rc=<1>
Feb  5 08:03:30 bigip2.local err dhcp_config[31186]: save error: 01070734:3: Configuration error: invalid Management Route, the dest/netmask pair 169
.254.169.254/255.255.255.255 already exists for /Common/azureMetadata
Feb  5 08:03:30 bigip2.local err NET[31190]: dhclient: dhcp-config failed with rc=<1>
Feb  5 08:08:54 bigip2.local notice NET[19857]: /sbin/dhclient-script : updated /etc/resolv.conf
Feb  5 08:08:54 bigip2.local notice NET[19870]: /sbin/dhclient-script : updated /etc/resolv.conf
Feb  5 08:09:06 bigip2.local notice vadc-init[20028]: vadc-init: Run all the deployment-specific startup scripts first
Feb  5 08:09:06 bigip2.local notice vadc-init[20030]: vadc-init: Initializing cloud-init
Feb  5 08:09:06 bigip2.local notice vadc-init[20042]: vadc-init: vadc-init - DONE
Feb  5 08:09:56 bigip2.local err dhcp_config[28211]: save error: 01070734:3: Configuration error: invalid Management Route, the dest/netmask pair 169.254.169.254/255.255.255.255 already exists for /Common/azureMetadata
Feb  5 08:09:56 bigip2.local err NET[28315]: dhclient: dhcp-config failed with rc=<1>

```

#### How do we fix this?

You can run the lines below to restore your mgmt IP.
```
tmsh create sys management-ip 10.0.0.11/24
save sys config
```

But to prevent this happening again, convert away from DHCP for your mgmt addresses, or avoiding overlapping static routes on DHCP interfaces.

### Key Files and Commands
- Files
  - `/var/lib/dhclient/dhclient.leases` *-> contains information about DHCP leases and includes routes*
  - `/var/log/boot.log` *-> contains errors when DHCP routes overlap static routes*
- Commands
  - `tmsh show sys ip-address | grep management-ip` *-> shows mgmt IP addresses on the box*
  - `tmsh list sys management-route` *-> shows all routes on mgmt interface, both configured by DHCP or within BIG-IP.*

### Summary
We learned a few important lessons here:
1. Do not use DHCP for mgmt IP addresses on network devices or your servers.
2. Avoid using static routes on dynamically-configured interfaces.
3. If you must, for some reason, use DHCP and static routes together, avoid overlaps!

<figure>
    <a href="/assets/dhcp-static-routes/network-engineers-harmony.webp"><img src="/assets/dhcp-static-routes/network-engineers-harmony.webp"></a>
    <figcaption>AI-generated image of network engineers sitting in harmony.</figcaption>
</figure>