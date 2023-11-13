---
layout: post
title:  "More on Azure LB and flow symmetry"
date:   2023-10-31 09:00:00 -0400
categories: Azure
---

Yesterday I wrote a [blog post]({% post_url 2023-10-30-azure-load-balancer-5-tuple %}) with some notes on Azure LB and thought out loud about Azure LB and flow symmetry. I still think it acts more like DAG than a network load balancer.

But after I finished writing I thought about a couple of ideas.

### Further thoughts about load-balancing firewalls in Azure

1. I _think_ I should have added another rule to obey when you want the holy grail of Active/Active, AND no SNAT. **And that rule is: no proxying, only routing (ie, no performing DNAT).** If you were to proxy the traffic at your F5 BIG-IP (or PaloAlto or whatever), how could Azure LB possibly know to match the server response traffic to the existing connection?

2. If the above is true, this architecture is often going to apply to firewalls but rarely F5 BIG-IP's, because BIG-IP's tend to perform DNAT by their very nature (load balancing). While you _can_ load balance to pool members with Destination NAT disabled, it relies on MAC addresses and therefore won't load-balance as required in public cloud environments.

3. Given yesterday's post and my thoughts above, I should probably write a dedicated post and/or a DevCentral article covering how, why, and gotchas for this architecture. Nicely cleaned up with some diagrams, this could be customer-facing and used by others.

4. I should search google for "how azure lb maintains flow symmetry with firewalls". This led me to this excellent document about highly available NVA's: [https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha)

### Highly available NVA's

[This document](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha) did an even better job than the [one I referenced yesterday](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/firewalls/) discussing how to use Azure LB to achieve HA between your firewalls where you can have that magic combination of **active/active AND no SNAT required**. 

#### Things to remember / gotchas:
* Azure LB ensures flow symmetry for you:
> For traffic between on-premises networks and Azure or between Azure virtual machines, traffic symmetry is guaranteed by the internal Azure Load Balancer: when both directions of a traffic flow traverse the same Azure Load Balancer, the same NVA instance will be chosen.
* So you can use Active/Active for your NVA's:
> This setup supports both active/active and active/standby configurations. However, for active/standby configurations the NVA instances need to offer a TCP/UDP port or HTTP endpoint that doesn't respond to the Load Balancer health probes unless the instance is in the active role.
* But, you must use a single interface to route the traffic. This one we learned from [documentation previously covered](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/firewalls/).
> Problems arise when we add another interface to the firewall, and you need to disable NAT translation between internal zones......<br/>
> (When using a single interface)...the iLB may match the traffic to the initial session, so it will always point this traffic back to FW-1. FW-1 then sends it to S1, establishing a synchronous traffic flow.
* Due to the Active/Standby model requiring a health check that does NOT respond when the device is Standby, we need a solution like the AS3 template I've linked to above.
* Any traffic that traverses the public Internet must be SNAT'd if you want to maintain flow symmetry. To quote the document:
> For traffic between Azure and the public Internet, each direction of the traffic flow will cross a different Azure Load Balancer (the ingress packet through the public ALB, and the egress packet through the internal ALB). As a consequence, if traffic symmetry is required, Source Network Address Translation (SNAT) needs to be performed by the NVA instances to attract the return traffic and avoid traffic asymmetry. 

#### Lastly, a nice diagram
The diagram below is pulled from the section of the article titled [Load Balancer Design](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/dmz/nva-ha#load-balancer-design) and while the author uses the same diagram to depict on-prem-to-AzureVNET traffic as VNET-to-VNET (because the architecture is exactly the same), this diagram can be used to explain to firewall admins that Active/Active, no SNAT firewall load-balancing is possible in Azure when you meet the criteria outlined above.

![image NVA's in High Availability](/assets/nvaha-load-balancer-on-premises.png)