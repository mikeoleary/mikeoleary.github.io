---
layout: post
title:  "Inbound SSLo and AWS notes - part 3"
categories: [aws]
tags: [aws,sslo]
description: Brief notes about AWS, SSLO, PA and Fortinet. These are rough notes I made during my first successful deployment of SSLo in AWS using GWLB and Fortinet.
---
This is 3-part blog post:
* [Part One]({% post_url 2023-11-19-aws-sslo-inbound-part1 %}) - intro 
* [Part Two]({% post_url 2023-11-27-aws-sslo-inbound-part2 %}) - rough notes taken during troubleshooting
* [Part Three]({% post_url 2023-11-30-aws-sslo-inbound-part3 %}) - working lab overview and conclusion

### Background
<!-- begin_excerpt -->
I started this series with [Part One]({% post_url 2023-11-19-aws-sslo-inbound-part1 %}), a nice overview of Inbound SSLo in Azure, as well as an explanation of why I would use GWLB in AWS and a diagram of a working GWLB + BIG-IP set up in AWS. I ended that post with a laundry list of follow-on tasks which are now all complete. This post will explains the process to get a working lab set up.

<!-- end_excerpt -->
### Setting up SSLo after GWLB
For reference, that diagram we left off with is below:
![AWS SSLo inbound](/assets/AWS-SSLo-inbound-1.png)

The missing part of the diagram above is the SSLo set up. In the diagram above, the BIG-IP is simply receiving traffic, optionally applying security controls such as WAF policies or L3/L4 or IPS security measures, and then sending the traffic back to the GWLB.

It is possible to proxy traffic out via another interface of the BIG-IP and have the traffic sent to another server or device, before sending the response traffic back through the Geneve tunnel. That's what we will do to achieve SSLo.

### Adding Fortinet as an inspection device
I don't know Fortinet products very well, but I fumbled through the set up and got this working by using the SSLo guided config (L3 inbound topology) and following these steps:
1. **Deploy a Fortinet device with 3x  NIC's**<br/>
   a. Eth0 for management console.<br/>
   b. Eth1 for receiving traffic from BIG-IP eth2<br/>
   c. Eth2 for forwarding traffic toward BIG-IP eth3
2. **I used 3x dataplane NIC's on my BIG-IP** (not including my mgmt interface on eth0)<br/>
   a. Eth0 for mgmt console.<br/>
   b. Eth1 for receiving traffic via the Geneve tunnel.<br/>
   c. Eth2 for SSLo to send traffic to the Fortinet device.<br/>
   d. Eth3 for SSLo to receive traffic from the Fortinet device.
3. **BIG-IP and Fortinet in same 2 subnets** <br/>
   a. Eth2 on BIG-IP and Eth1 on Fortinet are in same subnet<br/>
   b. Eth3 on BIG-IP and Eth2 on Fortinet are in same subnet<br/>
4. **Routes set up on Fortinet**<br/>
   a. Fortinet has 2 routes:<br/>
       i. One is to send traffic destined to the web servers out Fortinet eth2 toward IP address of BIG-IP eth3.<br/>
       ii. One is to send all other traffic (0.0.0.0/0) out Fortinet eth1 toward IP address of BIG-IP eth2.<br/>
5. **Disable Address Translation on BIG-IP**<br/>
    Make sure that you have SNAT set to None in your SSLo Guided Configuration wizard.<br/>
    a. If you are using "L3 inbound" as your topoly, there will be two places to set this. You need SNAT=none for your SSLo inspection devices, so that SNAT (and DNAT) is not applied to the unencrypted traffic toward the Fortinet. You also need SNAT=none for your egress from the BIG-IP, because you cannot change the Src IP address when you return the traffic through the Geneve tunnel.<br/>
    b. If you used an "existing application" topology, then just ensure that SNAT=none for your inspection device. Your SNAT setting for egressing BIG-IP will be under your existing Virtual Server.

### Other tips
* I made some troubleshootig notes in [Part Two]({% post_url 2023-11-27-aws-sslo-inbound-part2 %})
* In the last couple of steps of the SSLo wizard, you need to egress out the Geneve tunnel
* If you add/remove ENI's, always remember to disable the src/dest check in AWS
* Packet capture is your best troubelshooting tool
* AWS route tables were not used to direct traffic between BIG-IP and Fortinet. Just the routes on the devices. This doesn't work in Azure, where we used route tables, but does on AWS when devices are in the same subnet.

### Final working lab diagram
The diagram below extends our previous lab, which showed AWS GWLB + F5 BIG-IP only, to include SSLo configuration and Fortinet inspection devices. The numbered steps 1 through 6 show the general flow of traffic inbound, and then the response traffic will take the same path in reverse.
![AWS SSLo inbound](/assets/AWS-SSLo-inbound-2.png)

### Conclusion
After all this set up, I think it's worthwile that the benefit that GWLB is bringing to this lab is the ability for the BIG-IP and Fortinet devices to see true destination IP address of the web server being targeted, and the ability to run our devices Active/Active without needing to cluster them (in fact, this is better descrived as 2x standalone Active devices).

We could also see true destination IP by using an alien range (for internal traffic) or EIP's on the BIG-IP (for Internet-facing traffic) and still achieve HA (albeit, Active/Standby only).

To conclude, SSLo is complex to understand by relatively straightforward once implemented. The same is true for GWLB - it is difficult as a concept but it requires very little operational management after initial set up. So, reach out if you'd like to use SSLo in AWS and I can help you set this up if needed!
