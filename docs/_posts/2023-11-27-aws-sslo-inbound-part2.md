---
layout: post
title:  "Inbound SSLo and AWS notes - part 2"
categories: [aws]
tags: [aws,sslo]
description: Brief notes about AWS, SSLO, PA and Fortinet. These are rough notes I made during my first successful deployment of SSLo in AWS using GWLB and Fortinet.
---


<!-- begin_excerpt -->
Since writing my [first post]({% post_url 2023-11-19-aws-sslo-inbound-part1 %}) about setting up SSLo in AWS, I have come to deploy a security device in the service chain and thought to make a few notes.

1. In AWS, I've deployed a Fortinet Firewall with 2x data plane NIC's. (One in the same subnet as my SSLo egress NIC on BIG-IP, the other in the same subnet as my SSLo ingress NIC on BIG-IP.)

2. In the end, **I didn't use Route Tables in AWS to direct traffic**. Because the Fortinet and BIG-IP were in the same 2 subnets, I did this:

   a. Fortinet NIC 2 had a Static Route pointing 0.0.0.0/0 to BIG-IP's "SSLo egress" NIC IP address.

   b. Fortinet NIC 3 had a Static Route pointing 10.1.10.0/24 to BIG-IP's "SSLo ingress" NIC IP address. (This is the CIDR block that the destination IP addresses fall in as traffic traverses the Fortinet)

   c. Fortinet NIC 1 was dedicated for mgmt console, which I access via a jump host in the same subnet. 

3. Command to change admin password, I found it [here](https://live.paloaltonetworks.com/t5/general-topics/from-the-cli-can-i-update-other-admin-account-passwords/td-p/26668).
   ```
   configure
   #set mgt-config users admin password
   ```
<!-- end_excerpt -->

4. Deploy a Fortinet FW in AWS with multiple NIC's.

   [This link](https://community.fortinet.com/t5/FortiGate/Technical-Tip-How-to-disable-Reverse-Path-Forwarding-RPF-per/ta-p/193338) taught me this command, to disable anti-spoofing measures that were interfering with my testing.
   ```
   # config system interface
   edit <interface>
   set src-check disable
   end
   ```

5. [Another link](https://community.fortinet.com/t5/FortiGate/Technical-Note-How-anti-replay-works-and-sniffer-usage-for/ta-p/194182) for disabling anti-replay. I don't need to know what it is, I just disabled it to get a basic routing demo working.
   ```
   # config system global
   set anti-replay disable
   end
   ```

6. [This link](https://www.hifence.com/blog/fortigate/the-definitive-guide-to-fortigate-troubleshooting-cli/) explained how to show the route table for the whole device and some other commands. Since I don't know Fortinet products at all, this was helpful.
   ```
   # get router info routing-table all
   ```
