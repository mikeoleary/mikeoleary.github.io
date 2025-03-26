---
layout: single
title:  "Persistent IP address for my Hyper-V guest on Windows 11"
categories: [Hyper-V]
tags: [kubernetHyper-Ves]
excerpt: "After a few years of this annoying problem, I've finally fixed it." #this is a custom variable meant for a short description to be displayed on home page
gallery:
  - image_path: assets/hyper-v-switches/Hyper-V-Network-Switches-01.png
    url: assets/hyper-v-switches/Hyper-V-Network-Switches-01.png
    title: "Private"
  - image_path: assets/hyper-v-switches/Hyper-V-Network-Switches-02.png
    url: assets/hyper-v-switches/Hyper-V-Network-Switches-02.png
    title: "Internal"
  - image_path: assets/hyper-v-switches/Hyper-V-Network-Switches-03.png
    url: assets/hyper-v-switches/Hyper-V-Network-Switches-03.png
    title: "External"
gallery2:
  - image_path: assets/hyper-v-switches/windows_network_settings_1.png
    url: assets/hyper-v-switches/windows_network_settings_1.png
    title: "Private"
  - image_path: assets/hyper-v-switches/windows_network_settings_2.png
    url: assets/hyper-v-switches/windows_network_settings_2.png
    title: "Internal"
  - image_path: assets/hyper-v-switches/windows_network_settings_3.png
    url: assets/hyper-v-switches/windows_network_settings_3.png
toc: true
---
<figure>
    <a href="/assets/hyper-v-switches/linux-guest-hyper-v-networking.webp"><img src="/assets/hyper-v-switches/linux-guest-hyper-v-networking.webp"></a>
</figure>

### Summary
The default virtual switch in my Hyper-V installation is an Internal virtual switch that allows access to the host network via NAT. However, the DHCP range of this switch changes with every host reboot. I have finally got around to fixing an annoying problem I'll describe later. The fix: a custom internal virtual switch, a VM guest with a static IP, and a NAT for outbound internet access.

### Details
For 3 years I've had this pet peeve. 

I run a Windows 11 laptop, but every day I use an Ubuntu VM running locally on my laptop with Hyper-V. I'm sure there's other ways, but I like this setup: 
- it's cheap (no cloud compute costs)
- my VM is persistent
- I can [rebuild it with my favorite tools]({% post_url 2024-06-07-ubuntu-server-post-deploy-script %}) very easily.

#### Annoying problem
I use Ubuntu 22.04 because it's a quick to deploy. I attach the default virtual switch because it's easy. The VM would get an IP address via DHCP from this Internal switch, I'd connect to it via SSH from Windows, and I'd go to work.

However, the IP address of my VM would change with every reboot of the Hyper-V host. The Hyper-V host is my laptop, so it gets rebooted often. Of course we expect IP changes with DHCP, but the real annoying part: *the DHCP range of the default virtual switch was changing with every laptop reboot*.

This meant that after every laptop reboot, I had to open Hyper-V to see what the IP address of the guest VM was, and then connect to it. Given that I like to connect to it using tools like WinSCP, and sometimes VSCode, and sometimes Putty, I would update my saved connection in each of these tools if/when needed. This took a couple seconds, but it was annoying to do after every laptop reboot.

<figure>
    <a href="/assets/hyper-v-switches/changing-ip-address-in-hyper-v.png"><img src="/assets/hyper-v-switches/changing-ip-address-in-hyper-v.png"></a>
    <figcaption>See that when the host reboots, the VM gets a new IP address from a new IP range.</figcaption>
</figure>

[All](https://www.reddit.com/r/HyperV/comments/iy38ig/fixed_hyperv_virtual_switch_network/) [of](https://learn.microsoft.com/en-us/answers/questions/48268/change-hyper-v-(default-switch)-ip-address-range) [these](https://stackoverflow.com/questions/63449007/how-to-prevent-the-ip-address-of-hyper-v-virtual-switch-from-being-changed) cases seem to review the same annoying problem. 

##### Why not configure a static IP on my guest VM when using the default virtual switch? 
I could do this, but the default virtual switch's IP address range would change with every laptop reboot. So, after the next reboot, my guest's static IP address would fall outside of the range of the switch to which it was connected, making it unreachable. It wasn't just the changing IP address that was annoying, it was the changing IP range of the default virtual switch.

##### Why not use an external virtual switch?
An external virtual switch is assigned to a physical NIC on the host. My laptop has 2 physical NIC's and I use them both often: the built-in ethernet NIC, and the built-in Wifi NIC. Because it's a laptop, I switch between wired connections (eg, at my desk) and Wifi (when I travel, go to an event, or just work from the lounge room).

Let's answer this question considering both scenarios: external virtual switch assigned to ethernet NIC, and to Wifi NIC.

###### Ethernet NIC
This would work most of the time. An External virtual switch allows my host to access the guest VM, the guest VM can access the host network, and it would receive DHCP from my home's router. This means it would usually keep the same IP address (or I could make it static). The annoying part here is when I switch to Wifi, which is often enough to be a pain. I would have to either:
1. Edit the External virtual switch to be assigned to my Wifi NIC, or
2. Edit the VM to connect to a different External virtual switch that is permanently assigned to my Wifi NIC

I found that using the default virtual switch (an internal switch), allowed me to leave the virtual switch setting alone in Hyper-V when moving between wired and wireless networks.

###### Wifi NIC
Further, a problem with Wifi and DHCP: Given that it's normal for different Wifi networks to have different IP ranges (the coffee shop, the hotel, etc), I would have the same problem with Wifi: the DHCP-assigned IP address for my guest VM would change if using an external NIC, and I would then need to update my connection settings in various tools.

To summarize, I tried using an External virtual switch but still found it a hassle.

##### Why not use a private virtual switch? 
By now we know that Hyper-V has external, internal, and private virtual switches. [This page](https://www.bdrsuite.com/blog/hyper-v-network-switches-choosing-the-right-one-for-your-setup/#External) has a nice overview of the differences and nice diagrams. 

{% include gallery caption="Private, Internal, and External virtual switch types"  %}

Because a private virtual switch does not offer connection to the host network at all, this didn't suit me. I want to connect to my VM via SSH. I also want my guest VM to have outbound Internet access, but I don't need it to be directly accessed from the outside network.

##### Why not create a new Internal virtual switch?
That's what I did in the end! I didn't realize a few things until I took the time to read, which is why it took me so long to get around to it, and also why I'm creating this post:
- an Internal virtual switch does not allow outbound host network access by default
  - the default virtual switch is Internal, but does allow access to the host network via NAT.
  - to achieve the same outbound access to the Internet with an Internal virtual switch you deploy, you can create a NAT network in Windows that matches the IP range of your Internal virtual switch's range
  - DHCP is provided by the default virtual switch, but if you create your own Internal virtual switch, it won't serve DHCP requests. Use your own DHCP server or use static IP's for guest VMs.

##### What else could you do? 
Instead of creating a new Internal virtual switch with a static IP address, I could write a script to re-IP the default virtual switch after every reboot, like [this guy](https://urbanek.io/post/fix-hyperv-network/) did. It achieves the same thing (a persistent known IP address range for the virtual switch) but requires a PowerShell script run after *every* reboot and that *also* requires admin privileges. I like my way better, but this is still a nice illustration of our problem.

### How-to

#### Create a new virtual switch of type Internal
This is very straightforward because there's very few options. Once you've created a virtual switch, you'll see a new virtual network adapter (vEthernet adapter) assigned to the host server (my laptop), allowing the host to communicate with VMs on the Internal switch. 

<figure>
    <a href="/assets/hyper-v-switches/new-hyper-v-virtual-switch.png"><img src="/assets/hyper-v-switches/new-hyper-v-virtual-switch.png"></a>
</figure>

#### Choose an IP range for your new virtual switch
I've chosen a small range, hoping that it won't overlap with any Wifi networks I might join in the future: 172.20.100.0/24

If I do join a Wifi network that overlaps that range, I think I'll have problems because I'll have a route on my Windows host that sends this route via my Wifi NIC, and an overlapping route for my virtual network adapter. I suppose that's the cost of static IP assignment, but no DHCP-magic can overcome an overlap like this either and still keep me from having to update my saved connections in my tools.

#### Assign an IP address to this virtual network adapter
I've assigned 172.20.100.1 to the virtual network adapter, and given it a /24 mask. No default GW because I only want a single default gateway for my Windows laptop.

{% include gallery id="gallery2" caption="Windows network config steps"  %}

#### Assign a static IP address to the guest VM
This of course is unique to whatever OS your guest VM runs. Here's my netplan config for Ubuntu. (And a [netplan tutorial](https://linuxconfig.org/netplan-network-configuration-tutorial-for-beginners) if you're new to that).

<figure>
    <a href="/assets/hyper-v-switches/netplan-config-ubuntu.png"><img src="/assets/hyper-v-switches/netplan-config-ubuntu.png"></a>
</figure>


#### Finally, create a NAT network via PowerShell to allow outbound internet access.
I touched on this earlier but it's important to repeat: by default, a new custom internal virtual switch will not allow outbound access to the Internet for guest VM's, but your host can access guest VM's via the network. The result is that you have inbound access to your guest VM's, but they don't have outbound access. 

To allow your guest VM's outbound access via NAT, follow [these instructions](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network) to run this command as an administrator in PowerShell:<br/>

`New-NetNat -Name MyNATnetwork -InternalIPInterfaceAddressPrefix 172.20.100.0/24`

<figure>
    <a href="/assets/hyper-v-switches/Powershell-screenshot.png"><img src="/assets/hyper-v-switches/Powershell-screenshot.png"></a>
</figure>

### The final result
I now have what I want:
- a persistent Hyper-V guest VM for daily use
- a persistent, known IP address for this VM across laptop reboots
- network access from my Windows laptop running tools like Putty, VSCode, or WinSCP to my guest VM (possible only with External or Internal types of virtual switches)
- network access from my guest VM to the Internet (possible with External, or Internal with a NAT network, but not Private)
- the ability to switch between physical network adapters (ethernet and Wifi) and not need to edit anything in Hyper-V

### Feedback and summary
I like my persistent Ubuntu VM. I'm not a professional developer, but if I was I'd consider using a remote dev environment or better shared dev tooling. I don't think my approach would scale for a team. 

I don't specialize in virtualization or Hyper-V so I'm open to learning new ways of practical approaches to dev/engineering workspaces and environments. Thanks for reading!



