---
layout: single
title:  "Active/Standby BIG-IP's behind Azure LB with Floating IP (no DNAT) on Azure LB"
categories: [azure]
tags: [azure]
excerpt: "Here's the details for a specific BIG-IP set up behind Azure LB. " #this is a custom variable meant for a short description to be displayed on home page
---

This article covers an advanced scenario for running an Active/Standby BIG-IP pair behind Azure LB. Read and follow these instructions **all** of the following conditions are met:
1. You want to run 2x BIG-IP devices in Active/Standby configuration
2. You want to provide High Availability between BIG-IP devices using Azure LB
3. Your intention is to have your BIG-IP's Virtual Server ip address (VIP) be the same as the frontend ipconfig on the Azure LB.
4. You have checked the "Floating IP" checkbox on your Azure LB rule that forwards traffic to BIG-IP

#### Options for High Availability (HA) of Network Virtual Appliances (NVA's) in Azure
This article does not cover the advantages and disadvantages of different methods to achieve HA in Azure. Broadly speaking, there are multiple approaches:
- **Azure Load Balancer**. This approach uses a simple, L4 load balancer in Azure to disaggregate traffic across multiple devices.
  - Active/Standby, Active/Active, or multiple standalone devices are options here. 
  - _This article focuses on a scenario using Active/Standby BIG-IP devices behind Azure LB, when the "Floating IP" checkbox is checked on the Azure LB rule._
- **F5 Cloud Failover Extension (CFE)**. This automation approach uses software on the BIG-IP device to ensure High Availability across devices that are Active/Standby, without requiring Azure LB.
  - It can move IP addresses between Azure network interfaces (emulating Gratuitous ARP that happens on-prem)
  - It can update a route table to ensure the default route (0.0.0.0/0) for a subnet sends response traffic to remote clients back via the active BIG-IP.
  - It can update a route table to point an entire CIDR block dedicated for VIPs at the active device. This is also referred to as an "alien range" approach. 
- **DNS Load Balancing**. A simple method for High Availability between devices is using DNS and multiple standalone appliances. However DNS load balancing within a local site is not common (although GSLB is a common practice across geographic regions)
- **Other approaches**. Azure have a Gateway Load Balancer that can be used but requires advanced knowledge, and you might even consider BGP in some cloud scenarios. These are out of scope for this article.

#### Common Architecture of Azure LB with BIG-IP pair

The common/easiest architecture of running Active/Standby BIG-IP devices behind Azure LB looks like the following diagram.

Note that the BIG-IP devices are in Active/Standby mode, and Azure LB is simply a "dumb" Layer-4 traffic disaggregator. 

<figure>
    <a href="/assets/azure-lb-active-standby-floating/default-lb-config.png"><img src="/assets/azure-lb-active-standby-floating/default-lb-config.png"></a>
    <figcaption>Default LB architecture</figcaption>
</figure>

#### Taking a closer look at IP addressing with this common architecture

Let's take a look at where we configure IP addresses based on the most common approach:

<figure style="width: 1200px">
    <a href="/assets/azure-lb-active-standby-floating/default-lb-config-with-comments.png"><img src="/assets/azure-lb-active-standby-floating/default-lb-config-with-comments.png"></a>
    <figcaption>Default LB architecture with IP addresses and NATing depicted</figcaption>
</figure>

There's a few things here that sometimes confuse the first-time cloud admin:
- this solution requires 2x IP addresses for every Virtual Server on BIG-IP. 
  - You could create 2 separate VS's, each with 1 IP address. 
  - You could also create a single VS with 2x IP's using Shared Address lists, or your Virtual Address could be a /30 range. Either way, it's different than what you're accustomed to on-prem.
- this means IP addresses will get used up twice as fast as we're accustomed to multiple destination NAT's can sometimes confuse app owners (although normally a network admin has no problem understanding this)

#### When and why to use Azure LB's "Floating IP" option.

The [floating IP checkbox](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-floating-ip) on your Azure LB rule could be understood as telling Azure LB: "do not perform Destination NAT for this traffic". This is similar to F5's [nPath](https://my.f5.com/manage/s/article/K11116) (aka assymetric, or Direct Server Return) architecture.

<figure style="width: 1200px">
    <a href="/assets/azure-lb-active-standby-floating/lb-config-floating-ip.png"><img src="/assets/azure-lb-active-standby-floating/lb-config-floating-ip.png"></a>
    <figcaption>LB config with floating ip</figcaption>
</figure>

Why would you configure as above?

- No destination NAT at Azure LB can make overall IP addressing easier
- 1x IP Address on your BIG-IP VIP is more like on-prem config we are familiar with

##### How to configure BIG-IP when "Floating IP" is used

The previous option is an alternative approach, but it requires a semi-advanced workaround for Azure health checks. _It's important to understand this workaround and if you don't, just stick with the common approach outlined first in this article._

If you use Floating IP with your Azure LB rule, health probes from Azure LB will target the primary ipconfig on the Azure NIC. In BIG-IP, that's your Self IP. And your Self IP will always respond healthy if it is probed, evn on the Standby device (of course, port lockdown settings on Self IP's must allow health checks).

Put another way: your Standby BIG-IP will respond as healthy to Azure LB, and Azure LB will send data plane traffic to it. This will cause problems, so we must make our Standby BIG-IP "unhealthy" in Azure LB.

Enter VIP targeting and iRules. Do this:

1. Create LB rule on Azure LB sending traffic to the primary ipconfig on the dataplane NIC on the BIG-IP devices.
2. Configure a health probe for this rule. A HTTP health check with default settings is fine.
3. Create an iRule on BIG-IP:
```bash
when HTTP_REQUEST {
HTTP::respond 200 content "device is active"
}
```
4. Create a VIP called ```/Common/unroutable_vip``` and give it an IP address of ```255.255.255.254``` and attach the iRule from the previous step. This VIP will only be reachable on the Active device, and is not routable from outside of BIG-IP.
5. Create another iRule:
```bash
when HTTP_REQUEST {
virtual /Common/unroutable_vip
}
```
6. On BIG-IP Device 1, create a VIP with the same IP addresses as the Self IP. [This is allowed](https://my.f5.com/manage/s/article/K13896). Listen on port 80, add HTTP profile, and attach the above iRule.
7. On BIG-IP Device 2, notice that the VIP created in the above step is not sync'd to Device 2. Repeat the above step with a VIP created on the same IP address as the Self IP. 

Now, your Azure LB will health check both devices, sending HTTP health checks to both devices and hitting the VIP's you created on the Self IP's. However, only the active device will successfully forward traffic to this VIP called "unroutable" and the Standby device will fail to do this. This means that Azure LB will believe that the Active is health and the Standby is down.

Don't forget you'll need the ```Enable IP Forwarding``` checkbox checked on your BIG-IP's interfaces in the Azure portal. 

##### Conclusion

If all of the above makes sense, feel free to use Azure LB's Floating IP checkbox on your Azure LB rules so that you can have the benefits of our last diagram. They are operational benefits only (fewer IP addresses used, potentially easier to understand for operators) but there is no functional or performance benefit to this method (no performance benefits or features/functionality enabled).

Thanks for reading, and please ask questions via comments or message me directly using this website.