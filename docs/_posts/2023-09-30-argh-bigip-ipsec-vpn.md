---
layout: post
title:  "Argh. BIG-IP IPSEC VPN to AWS TGW"
categories: BIG-IP
---

Sorry in advance. This is a rushed post intended for rough note taking for myself.

### Argh, IPSec VPN
Typically I don't like to use appliance-based VPN connections for the obvious reasons
- your appliance needs to be maintained
- it's vendor-specific
- IPSec VPN's are somewhat legacy in nature. 

So this post is intended to be very short and only for the purpose of me remembering how I did this.

### Set up a BIG-IP somewhere
- I used Azure in my demo.
- My VNET is 10.0.0.0/16
- I created another Ubuntu VM in Azure to act as a client

### Set up a AWS TGW with a Site-to-Site VPN connection using IPSec
- Create a VPC in AWS. Create a Ubuntu VM in AWS to act as a server. Install docker and a simple web app.
- [This page](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html) describes a Site-to-Site VPN in AWS
- Create a AWS TGW. Attach it to VPC.
- Create a TGW attachment. 
  - Attachment type is "VPN"
  - Create a new Customer Gateway (CGW) or use an existing one. IP address is public IP address of BIG-IP in Azure
  - I used Static Routing (not BGP) so ASN didnt matter
  - I left Inside IP CIDR for Tunnels to be generated by Amazon.
  - I used a pre-shared key

- Create a Site-to-Site VPN connection in AWS. 
  - Target gateway type is Transit Gateway (not Virtual Private Gateway)
  - Customer Gateway is the CGW set up earlier

### Configure BIG-IP
- basically following [this article](https://f5-agility-labs-public-cloud.readthedocs.io/en/latest/class2/module6/lab1.html), download the details of the S2S VPN connection so that they can be imported into BIG-IP.
- after doing this, edit a few things on BIG-IP
  - the local address on the tunnel in BIG-IP config should be the self-IP of the relevant Interface, probably not the public IP of the BIG-IP
  - I also set up a route on the F5 device so that the CIDR block of the AWS VPC was routed over the tunnel.
  - I also edited the IKE peer of the AWS TGW tunnels in BIG-IP config so that NAT Traversal was set to ON (not OFF). I had to do this via TMSH with the command below because the GUI told me that there was a general database error when I tried to edit the IKE peer in the GUI.

{% highlight bash %}
tmsh modify net ipsec ike-peer <tunnel name> nat-traversal on
{% endhighlight %}

### Route tables between clouds
- In Azure, set a UDR so that the CIDR block of AWS VPC is pointing at selfIP of BIG-IP. Check "IP Forwarding" enabled.
- In AWS, have a route table so that CIDR block of Azure VNET points at TGW.
- In AWS TGW, check the **Transit Gateway Route Table** that is associated with the Transit Gateway you are concerned with. Check the **Routes** tab to see how  the TGW will route traffic destined for the Azure VNET CIDR block.

### Test
From your host in Azure, curl or ping your host in AWS. 
- remember to allow HTTP and ICMP through your AWS Security Groups.
- remember you'll need a forwarding VIP created on BIG-IP, if the script didn't create one for you when you imported the configuration that you downloaded from AWS.

### Troubleshooting
- Command to see IPSEC tunnel on BIG-IP. You want it to have a status of Created with a timestamp.
{% highlight bash %}
tmsh show net ipsec ipsec-sa all-properties
{% endhighlight %}


