---
layout: post
title:  "Azure LB and a few advanced notes"
date:   2023-10-30 09:00:00 -0400
categories: Azure
---

I deal with Azure Load Balancer a lot, and occasionally the Azure Gateway Load Balancer (GWLB) and, once in a while, other PaaS offerings like Application Gateway. I've written a handful of articles over the last few years that focused on or included Azure Load Balancer. The most popular seem to be these 2:

- [Practical consideritions for Azure Load Balancers](https://community.f5.com/t5/technical-articles/practical-considerations-for-using-azure-internal-load-balancer/ta-p/291195)
- [BIG-IP integrations with Azure Gateway Load Balancer](https://community.f5.com/t5/technical-articles/big-ip-integration-with-azure-gateway-load-balancer/ta-p/291102)

The following is mostly for the sake of my own bookmarking these sites. 

### Sites to remember when explaining Azure LB

#### 1. Floating IP #### 
In this mode, the LB does **not** perform Destination NAT as the traffic arrives at the VM. However, you **must** use the primary ipconfig (in F5 parlance, the self IP) as the backend pool member in the Azure LB. I never knew where this was documented before now, [but it's here](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-floating-ip#limitations).

> You can't use Floating IP on secondary IP configurations for Load Balancing scenarios. 

For this reason, if you want to use Floating IP and put F5 devices in Active/Standby in the backend pool, I suggest a workaround that I will blog about some other time, but you'll need to make sure the standby device is DOWN in Azure LB. The AS3 template I use is here: [https://github.com/mikeoleary/azure-aks-kic-cis/blob/master/infra/bigip/baseline-as3.json](https://github.com/mikeoleary/azure-aks-kic-cis/blob/master/infra/bigip/baseline-as3.json)

#### 2. Achieving flow symmetry without SNAT'ing between client and server
After what feels like months, I have finally I have found [this Microsoft article](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/firewalls/) that shows flow symmetry can be achieved in the following scenario:
- Azure LB used with 2 or more Active/Active firewalls behind it
- firewalls have a single NIC that is behind Azure LB that is serving both client->server and server->client traffic.
- client subnet and server subnet both have routes forcing traffic via the Azure LB IP address.

This lovely article again: [https://learn.microsoft.com/en-us/azure/architecture/example-scenario/firewalls/](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/firewalls/)

I ended up [submitting some feedback](https://github.com/MicrosoftDocs/architecture-center/issues/4281) because I was so happy to find this article and I think it deserves some clarification.

#### 3. Active/Active Palo Alto firewalls guide

[This guide](https://www.paloaltonetworks.com/apps/pan/public/downloadResource?pagePath=/content/pan/en_US/resources/guides/azure-transit-vnet-deployment-guide) shows how to set up Active/Active Palo Alto fw's. My interest is in pages 21-75, where E-W security is implemented between 2 spoke VNETs (no SNAT'ing). This shows a single Azure LB with 2x active FW's behind it which are inline between 2 spoke VNETs. Both spoke VNETs have default routes pointing to the Azure LB. The devices are not configured for DNAT or SNAT between VNETs. This means that we have a reference architecture where stateful devices that are routing, not proxying (ie not performing DNAT or SNAT), are running active/active behind Azure LB. 

This is also the model that I use when I suggest an inbound SSLO architecture [in this DevCentral article](https://community.f5.com/t5/technical-articles/sslo-in-public-cloud-azure-inbound-l3-use-case/ta-p/318351).

#### 4. Matching traffic flows

For this to work, Azure LB must match traffic flows, so that 'request' traffic that is client->server will traverse the same firewall as the 'response' traffic that is server->client. 

As we know, [Azure LB uses a 5-tuple hash](https://learn.microsoft.com/en-us/azure/load-balancer/concepts) to load-balance traffic to backend servers. This means that a single TCP connection will persist to the same backend service without the need for any stickiness. However, if the same client establishes a new session to the same server, the random src port will change, and therefore the hash of the 5-tuple. This is why Src IP and Src IP+ Protocol are offered as persistence types in Azure LB. 

What I'm interested in is *how* Azure LB is matching the 'response' traffic, or the flows that are server->client. The *components* of the 5-tuple hash are the same (src IP, src port, dest IP, dest port, protocol), but the *order* is now different (because src IP is now dest IP, and src port is now dest port) which means the *5-tuple hash must be different*, right? I have not found the answer documented anywhere, but the Microsoft article and Palo Alto reference architecture above rely on it. 

It's as if the Azure LB acts more like a DAG (disaggregator) than a network load balancer. Is that a fair comparison? That's a genuine question, I really don't know.

I've also noticed that flow symmetry is not maintained if the Azure LB pool members have multiple NIC's in use. This is also noted in the Microsoft articles above. Ie., if you have traffic LB'd to eth1 of a PaloAlto firewall or BIG-IP, and then egress eth2 and flow toward a server, even if you have a LB frontend in front of your eth2 NIC's in Azure, your response traffic might get LB'd to the "other" firewall or F5 BIG-IP.

In that sense, this flow symmetry only seems to be _per Azure LB frontendipconfig_. When I read some of F5's [DAG modes](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/bigip-service-provider-generic-message-administration-13-0-0/5.html), I see that 
> DAG is configured per VLAN. Note, this means that the client and server sides of BIG-IP should be configured on different VLANs. So it's possible to configure different DAG modes for client and server connections. **However, when a server responds to a client request, and a connection is already established, DAG is not used**.

The bold highlighting in the blockquote above is mine. I wonder if this is similar to the reason that flow symmetry is maintained by Azure LB for server response traffic. Perhaps 'DAG' is not used when server response matches an existing client connection. From using F5 BIG-IP or any firewall, we know that "flow symmetry" is maintained even when multiple NIC's are in use for client->server traffic, but perhaps there's limitations under the hood with Azure LB that mean that flow symmetry is only maintained if a single frontendipconfig is in use. 

Anyway, to summarize: Based on Microsoft's article we know that flow symmetry can be maintained by Azure LB when 2 or more firewalls are Active/Active and have no SNAT configured, as long as all client<->server traffic traverses the same Azure LB frontendipconfig, "Floating IP" is checked, and the firewalls are configured with a single dataplane NIC.







