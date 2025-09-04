---
layout: single
title:  "Unmanaged disks in Azure"
categories: ["big-ip"]
tags: ["big-ip"]
excerpt: "Unmanaged disks are finally going away. Remember those?" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/azure-unmanaged-disks/azure_disk_header.svg"><img src="/assets/azure-unmanaged-disks/azure_disk_header.svg"></a>
</figure>

### Summary

Recently I had a customer ask how to migrate BIG-IP's from unmanaged to managed disks in Azure. This is an overview.

### Background

Before Azure had managed disks, we had to use vhd files that were in blob containers in storage accounts. I remember doing this around 2016 before the introduction of managed disks. Managed disks were a big relief because you didn't have to worry about 
- the replication configured for storage accounts
- the performance of your storage account
- whether you had multiple busy VM's with their disks in the same storage account
- etc.

Now in 2025, it's hard to remember a time with unmanaged disks. Today I was made aware that they still exist, but [will be deprecated by March 31st, 2026](https://learn.microsoft.com/en-us/azure/virtual-machines/unmanaged-disks-deprecation).

### What about F5 BIG-IP?

I don't think there's anything special about the vendor of a VM. I started at F5 around Dec 2018, and I think we had managed disks already in use by then in our Azure ARM templates. 

But sure, you can test this out for yourself. First, create a BIG-IP with an unmanaged disk, and then migrate it to managed disk.

#### How to deploy a BIG-IP with an unmanaged disk

It's a hassle to actually deploy a BIG-IP with an unmanaged disk today because, as these [instructions](https://clouddocs.f5.com/cloud/public/v1/azure/Azure_download.html) indicate, the .vhd file you download from [https://downloads.f5.com](https://downloads.f5.com) must be used as the source blob for creating an Azure custom VM Image object. When you use that Image to deploy a VM, Azure will create a VM with a managed disk (not an unmanaged disk).

Perhaps you're thinking, can I just create an Azure VM with the .vhd file I downloaded as an "attached OS disk"? That will deploy a VM and you'll be able to reach it via SSH and GUI (on tcp/8443), but you won't be able to log in. There's no "admin/admin" or "root/default" user. You need to create an Azure custom image, and then deploy a VM from that.
{: .notice}

So, we need to start with a .vhd file of a BIG-IP that has actually been created for deployment of VM's, which you can do by using the (now deprecated) F5 image bakery tool, or by stopping a running VM and downloading the disk as a VHD file.

Here's what I did to get a BIG-IP with an unmanaged disk:
- deployed an Azure VM with a managed disk (using GUI, ARM template, whatever method you like)
- stopped that running BIG-IP and then download the raw VHD from the managed disk
- uploaded that VHD to a blob container within a storage account
- created a VM with an attached disk (example below)

```bash
az vm create \
--resource-group moleary-disk-test-rg \
--name f5testvm \
--location eastus2 \
--use-unmanaged-disk \
--os-type linux \
--attach-os-disk https://<storangeaccountname>.blob.core.windows.net/<containername>/<blobname.vhd>
```

This will create a VM in Azure, with VNET and a public IP. The OS disk will be the blob file and it will appear to the VM as a scsi disk. But of course this should not work after March 2026!

<figure>
    <a href="/assets/azure-unmanaged-disks/unmanaged-disk-vm.png"><img src="/assets/azure-unmanaged-disks/unmanaged-disk-vm.png"></a>
    <figcaption>Now we have a VM with an unmanaged disk. The source of the OS disk is a blob in a container.</figcaption>
</figure>


#### How do I convert my BIG-IP to a managed disk? 

Microsoft have [migration instructions](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/migrate-to-managed-disks) and an [FAQ](https://learn.microsoft.com/en-us/azure/virtual-machines/faq-for-disks?tabs=azure-portal).

The instructions and FAQs are worth reading for prod workloads. For example, your original VHD's in storage accounts don't get removed. So clean them up after.

```bash
#stop, convert, and start the VM
az vm deallocate --resource-group moleary-disk-test-rg --name f5testvm 
az vm convert --resource-group moleary-disk-test-rg --name f5testvm 
az vm start --resource-group moleary-disk-test-rg --name f5testvm 
```

Now let's check out our VM and it's disk again:

<figure>
    <a href="/assets/azure-unmanaged-disks/managed-disk-vm.png"><img src="/assets/azure-unmanaged-disks/managed-disk-vm.png"></a>
    <figcaption>Now we have a managed disk, with it's own properties, in Azure.</figcaption>
</figure>

You can see that if we click on the same VM and look at it's disk, the object is now a 'first-class citizen' in Azure. It has properties, tags, and settings that apply to the disk. The disk is not simply a .vhd file that resides in a storage account.

I can access the VM again with my chosen username and password. I know this is my original VM disk (and not a generalized OS image that's just been deployed.)

That original vhd file - the unmanaged disk - does exist in the storage account though, so don't forget to clean it up.

### Conclusion

1. If you still have unmanaged disks, you'll need to migrate them to managed disks in Azure by March 31, 2026. 
2. This will not be visible to your VM's, although will require a power down and boot up. Automate this as usual with preparation, logging, starting with pre-prod environments, and orchestrating your rolling outage.
3. Don't forget clean up.






