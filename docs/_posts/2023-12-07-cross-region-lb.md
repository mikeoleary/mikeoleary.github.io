---
layout: post
title:  "Using Azure's Cross-region Load Balancer with F5 BIG-IP"
categories: azure
tags: azure
description: "This post covers the new Azure Cross-region load balancer." 

---
![This pretty much explains the Azure Cross-Region load balancer](/assets/azure-cross-region-load-balancer.png)

    ^^ *This diagram above pretty much explains what this post is about* ^^

<hr />
<br/>
### Azure Cross-region load balancer

<!-- begin_excerpt -->
Recently I had a customer ask about when and why to consider Azure's [Cross-region load balancer](https://learn.microsoft.com/en-us/azure/load-balancer/cross-region-overview). Here's a few notes about this Azure service and the advantages and limitations compared to other approaches.
<!-- end_excerpt -->

From here on out, I'll refer to Cross-region load balancer as CRLB.

1. **Anycast IP** routing and the concept of a **global backbone** are pre-requisite knowledge. That's what makes this all work, so it's really nothing new in terms of technology. If you know what these 2 terms are, you can pretty much understand how a CRLB works.

2. **Load balancer of load balancers**. Basically a CRLB is a LB in front of a regular Azure LB. These regular Azure LB's are referred to as 'backend regional load balancers' in the MS documentation.

3. **Home Regions**. The concept of Home Regions isn't knowledge you need for most Azure deployments, and it really doesn't matter much. You choose a Home region to deploy your CRLB into, but it doesn't affect how traffic is routed. Still, you might have policies or other concerns about the logical home of your resources, so the supported Home regions are listed [here](https://learn.microsoft.com/en-us/azure/load-balancer/cross-region-overview#home-regions).

4. **Participating regions** is where the Global Public IP is advertised. I think of these as my physical locations of my Anycast IP address. They are listed [here](https://learn.microsoft.com/en-us/azure/load-balancer/cross-region-overview#participating-regions), and include most Azure regions I can think of.

5. **Other regions**. Your 'backend regional load balancers' can live in any publicly available Azure region. 

6. **Public Only.**. At least at the time of me writing this blog (it's the night of Nov 28, 2023), these are the biggest limitations that jump out at me:
   >Cross-region frontend IP configurations are public only. An internal frontend is currently not supported.
   and
   >Outbound rules aren't supported on Cross-region Load Balancer. For outbound connections, utilize outbound rules on the regional load balancer or NAT gateway.

### F5 BIG-IP deployments with Azure Cross-region load balancer

I can't see a whole lot changing right now for the average customer. Most F5+Azure customers have BIG-IP vm's running behind a regular Azure LB, but only public-facing LB's would consider tacking on CRLB.

I see 2 main reasons someone would do this:

1. **An alternative to Global Server Load Balancing (GSLB)**.

   GSLB isn't horrible and it's still widely used today. DNS failover has some annoying issues if you use it for internal clients or failover frequently, but for public sites, I'd still use it over CRLB today. Given the limitations of Azure's Load Balancers, there will be confusion for many using this.

2. **Performance for websites used by a globally diverse user base, especially where some users may have slow links or many ISP's to traverse.** Why? *Your pubic IP address will exist in many locations*. Every user will be sent to the closest Azure participating region (that's how Anycast IP routing works), and from there, they will traverse the Microsoft global network to the region where the 'regional load balancer' exists. It's a safe bet that traversing Microsoft's global backbone will be faster than traversing the public Internet.

   You might think, *"But with GSLB you can also send users to their closest site using geo topology!"*. True, but GSLB will only send users to sites where you have a public IP address configured. Most companies that I see use GSLB for HA between 2 regions. But Microsoft has 22 participating regions from which your public IP will be advertised!
   
   It's unlikely you would use GSLB to do better than this. (A more likely alternative would be to use your own global anycast network, or another provider, like F5 Distributed Cloud, or others.)

   ![Faster performance using Anycast IP Routing](/assets/azure-crlb-global-region-view.png)

#### Conclusion
This feature is relatively limited in terms of enterprise requirements. I'd start with new deployments and leave existing apps as they are architectured. I'd use a platform with Anycast built in (I am biased and prefer F5 Distributed Cloud) to build new apps and work toward HA/failover/performance improvements using Anycast. 

But for Azure customers that currently use GSLB between multiple existing public Azure load balancers purely for HA, the Azure Cross-region load balancer makes a lot of sense.