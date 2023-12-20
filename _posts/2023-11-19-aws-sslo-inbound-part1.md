---
layout: single
title:  "Inbound SSLo and AWS notes"
categories: [aws]
tags: [aws,sslo]
excerpt: My first article on F5 SSLo deployed in AWS.
---
This is 3-part blog post:
* [Part One]({% post_url 2023-11-19-aws-sslo-inbound-part1 %}) - intro 
* [Part Two]({% post_url 2023-11-27-aws-sslo-inbound-part2 %}) - rough notes taken during troubleshooting
* [Part Three]({% post_url 2023-11-30-aws-sslo-inbound-part3 %}) - working lab overview and conclusion

### Background
<!-- begin_excerpt -->
Recently I wrote on article in F5's DevCentral website: [SSLO in public cloud: Azure inbound L3 use case](https://community.f5.com/t5/technical-articles/sslo-in-public-cloud-azure-inbound-l3-use-case/ta-p/318351)

It was a great experience! My customer was a large hospital chain and they were able to follow the article to implement their cloud architecture, and then 2 other enterprise-level customers adopted the architecture after in the weeks after I wrote this article. 
<!-- end_excerpt -->

One of these customers was directed to my article from someone outside of F5. They decided to implement this and then reached out to me. This is my dream scenario for every article: to actually help real people and hopefully drive more adoption of F5 technologies too. 

![Azure SSLo inbound](/assets/azure-sslo-inbound-1.gif)

Inevitably, I have now been asked to do this same thing in AWS. So this blog post will serve as some initial notes, because it's going to be different.

### This has been done before
I haven't invented anything new here. My colleagues have written 2 articles that I'm basically going to copy:
* [Increase Security in AWS without Rearchitecting your Applications - Part 2: Wednesday Morning](https://community.f5.com/t5/technical-articles/increase-security-in-aws-without-rearchitecting-your/ta-p/307459) - *this is part 2 of a 4 part series by Heath Parrot*
* [Ingress/Egress VPC inspection with BIG-IP and GWLB](https://community.f5.com/t5/technical-articles/ingress-egress-vpc-inspection-with-big-ip-and-gwlb/ta-p/290792) - *this is an article that deals with AWS GWLB (not necessarily running SSLo) by Yossi Rosenboim*

Why am I even going to write another article? I am not sure I will, but if I do it will because I want a nice, simple article that is almost identical to the Azure one I referenced at the beginning of this blog. 

Heath's article series is detaild, and Yossi's article is great intro to GWLB and F5. But Heath's articles are deep and some folks like a quick read. And Yossi's demo code is now out of date and requires updates. So I don't think my article will confuse or overlap too much.

### What's different about AWS and Azure for F5's SSLo?
1. **In AWS we will use GWLB**. Why? Because in with AWS LB options, this is the only one that will *not* perform Destination NAT'ing, which is something I want in this scenario. In Azure I can use a regular Azure LB (not a Azure GWLB) because I can disable Destination NAT'ing with the Floating IP option. 

    We aren't *forced* to use GWLB. SSLO in AWS was possible before GWLB was released. We could use multiple ELB's but you'd either need a ELB per site (ie, ELB sprawl) or use L7 routing (host header or SNI-based routing). For HA, we could go Active/Active, remap EIP's, or update a route table like in [my still-popular DevCentral article](https://community.f5.com/t5/technical-articles/deploy-big-ip-in-aws-with-ha-across-az-s-without-using-eip-s/ta-p/291221).

2. **This means we'll have to use tunnels**. AWS GWLB uses Geneve tunnels. In Azure we didnt need to worry about tunnels. (Azure's GWLB uses VXLAN tunnels but that is out of scope.). 

3. **Azure does have a GWLB, but we cannot use it for SSLo**. Why not? Because Azure's GWLB must be attached to a regular Azure LB or a VM's IP address. So you have to target something else and then traverse the GWLB on the way. With AWS, we can send traffic to a GWLB and then it can follow a routing table toward it's next destination. We don't *need* another backend pool member in AWS.

### What's similar between AWS and Azure for SSLO?

1. **Route tables** are what you need to understand to make it all work. The complex stuff (tunnels, endpoints) is abstracted for you. The tunnel set up on your network virtual appliances is usually a once-off set up. But understanding what a route table is in Azure and AWS will take the customer most of the way.

2. In both AWS and Azure, if you are using a GWLB, your "pool members" behind the GWLB can be in a different VPC (AWS) or VNET (Azure). To reiterate, we will not use GWLB if we are doing SSLO in Azure, but it's worth knowing that both providers' GWLB offerings allow cross-network load-balancing without peeering the networks.

### What does a working AWS lab look like? 
Here's what I've done using Lucidchart so far:

![AWS SSLo inbound](/assets/AWS-SSLo-inbound-1.png)

Traffic flows from top to bottom. From the internet client (broswer), the inbound traffic hits the public IP address attached to my web server. The Internet Gateway (IGW) has a route table attached via an [edge association](https://docs.aws.amazon.com/vpc/latest/userguide/internet-gateway-subnet.html).

### Next steps
The above lab was set up by running the [Terraform demo](https://github.com/f5devcentral/f5-digital-customer-engagement-center/tree/main/solutions/security/ingress-egress-fw) that Yossi Rosenboim linked to from his [article above](https://community.f5.com/t5/technical-articles/ingress-egress-vpc-inspection-with-big-ip-and-gwlb/ta-p/290792). So, next steps:

1. Get SSLo set up. Right now this lab just has the F5 device to forward all traffic without applying anything (do Dest NAT, no Src NAT, no port translation, no security or iRules or traffic policies at all). 
2. Walk my customer through getting their Dev environment set up. They need help following the demo and getting a basic PoC together.
3. Potentially set up a Fortinet firewall as an inspection device. This will come in handy when my customer needs help, since they use Fortinet.
4. Potentially update this lab diagram.
5. Potentially write another DevCentral article for the AWS inbound SSLo solution.

