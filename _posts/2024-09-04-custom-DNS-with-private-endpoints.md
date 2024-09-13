---
layout: single
title:  "F5 CFE, private endpoints, and custom DNS"
categories: [azure]
tags: [azure, f5]
excerpt: "Recently a customer had a trifecta of conditions that forced me to think of a new solution, so I thought to document this." #this is a custom variable meant for a short description to be displayed on home page
---
### Summary
When using the F5 Cloud Failover Extension (CFE) for API-based failover in public cloud, some customers disallow API calls out to the public Internet. Private endpoints solve for this constraint, but rely on DNS. If a customer is using custom DNS servers, a workaround may be required.

### F5 CFE
The [CFE](https://clouddocs.f5.com/products/extensions/f5-cloud-failover/latest/) is code that runs on top of BIG-IP to perform failover in public cloud. It makes a series of API calls to the cloud provider to move IP addresses between interfaces, update public IP mapping between private IP's, or update route tables. It is supported in Azure, AWS, and Google Cloud (GCP).

<figure>
    <a href="/assets/cfe-custom-dns-private-endpoint/azure-failover-3nic-multiple-vs-animated.gif"><img src="/assets/cfe-custom-dns-private-endpoint/azure-failover-3nic-multiple-vs-animated.gif"></a>
    <figcaption>Typical case of CFE: API calls to update cloud configuration at failover. <a href="https://clouddocs.f5.com/products/extensions/f5-cloud-failover/latest/userguide/overview.html">Source.</a></figcaption>
</figure>

### Customer problem statement
My customer was using CFE to successfully move IP addresses between interfaces in Azure. They had followed default documentation and had a storage account that was publicly accessible, but only writeable by the BIG-IP VM's in Azure. They decided to create a private endpoint for their storage account in Azure and block Internet access to their storage account. 

CFE stopped working and reported "403 Forbidden" errors in logs. The issue was their use of custom DNS settings!

### CFE in isolated environments
Because the API calls are made to the cloud provider, the destination of these calls is a public API endpoint. For example:
- In Azure, storage API calls may be sent to `https://xxx.blob.core.windows.net`
- In AWS, storage might be accessed at `https://s3.us-east-2.amazonaws.com`
- In GCP, storage calls might be made to `https://storage.googleapis.com`

Since the BIG-IP is already running in the cloud provider, it's possible to access these services without traversing the Internet. Private endpoints allow a VM to access services within these cloud providers directly, and they can be set up in [Azure](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview), [AWS](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html), and [GCP](https://cloud.google.com/vpc/docs/private-service-connect).

[Documentation](https://clouddocs.f5.com/products/extensions/f5-cloud-failover/latest/userguide/isolated-env.html) exists to help F5 customers configure private endpoints in Azure and GCP. A DevCentral [article](https://community.f5.com/kb/technicalarticles/using-vpc-endpoints-with-cloud-failover-extension/278619) also walks through this for AWS users.

### Azure, Custom DNS, and CFE in isolated environments
My recent customer was running CFE in Azure, but this should apply to other cloud providers also.

Private endpoints work by using DNS. Once a private endpoint is configured, DNS entries are added for the networks that will use the private endpoint. 

For example, if I have an Azure storage account called `olearystorageacct`, blobs in this storage account will be reachable at `olearystorageacct.blob.core.windows.net`. This might resolve to a public IP of `1.1.1.1`. If I disable public Internet access to my Storage Account and create a private endpoint for it, VM's in my VNET will now resolve this name to a private IP address from their own VNET, such as `10.1.1.5` in the diagram below.

<figure>
    <a href="/assets/cfe-custom-dns-private-endpoint/azure-private-endpoint-storage-example.jpg"><img src="/assets/cfe-custom-dns-private-endpoint/azure-private-endpoint-storage-example.jpg"></a>
    <figcaption>An example diagram of a private endpoint for Azure storage account access.</figcaption>
</figure>

However, **this assumes VM's are using the default DNS settings in the Azure VNET**. When using custom DNS, VM's will resolve DNS queries against configured servers. In the diagram below, 10.0.0.100 and 101 are IaaS DNS servers.

<figure>
    <a href="/assets/cfe-custom-dns-private-endpoint/custom-dns-lookup-flow.png"><img src="/assets/cfe-custom-dns-private-endpoint/custom-dns-lookup-flow.png"></a>
    <figcaption>Example flow of DNS queries when custom DNS servers are used in VNET settings</figcaption>
</figure>

#### Working with custom DNS settings and private endpoints
Custom DNS represents a problem here. The IaaS DNS servers will forward DNS queries out to Internet-based DNS servers for domains like `blob.core.windows.net`. Despite the existence of a private endpoint, VM's using custom DNS may still resolve this record to a public IP address. How can we overcome this? 

I quickly found [others with the same problem](https://www.reddit.com/r/AZURE/comments/18sq80q/private_endpoint_for_virtual_network_with_custom/), and read related Microsoft [documentation](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview) that got my mental wheels turning.

I came up with this list of options for my customer, ordered from easiest to hardest in my opinion:

1. [Set a hosts record](https://my.f5.com/manage/s/article/K13206) on your BIG-IP device, so that `olearystorageacct.blob.core.windows.net` is manually set to the IP of the private endpoint, e.g. 10.1.1.5
2. [Set the DNS server of BIG-IP](https://my.f5.com/manage/s/article/K13205) to Azure's DNS server at 168.63.129.16. However, this means every DNS request from this VM will use Azure's DNS and not custom DNS servers.
3. Create a forwarding rule on the custom DNS server to forward requests for `*.blob.core.windows.net` to 168.63.129.16.
4. Create an A record on the custom DNS server: `xxx.blob.core.windows.net` could be set to 10.1.1.5
5. Lastly, a customer could always move away from custom DNS, and revert to default Azure settings for DNS. They can still use private DNS zones and Azure's [Private DNS Resolver](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/azure-dns-private-resolver). This is a larger architectural change of course, not a simple workaround.

My customer chose Option 2. CFE started working again! Their BIG-IP VM is using Azure's DNS server and accessing the storage account via private endpoint, but the custom DNS servers are in place for all other VM's following the VNET settings.

### Conclusion
In retrospect, this was a simple solution. When using both private endpoints and custom DNS together, plan for how DNS resolution will work for given services. I hope this article helps if you find yourself in the same situation!

