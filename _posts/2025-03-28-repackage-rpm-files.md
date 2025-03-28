---
layout: single
title:  "Repackage RPM file from Zstandard to Gzip compression"
categories: [linux]
tags: [linux]
excerpt: "Quick notes after I recently had to repackage an RPM file for an older RPM" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/repackaging-rpm-files/repackaging-rpm-files.png"><img src="/assets/repackaging-rpm-files/repackaging-rpm-files.png"></a>
</figure>

### Summary
AzureHSM support for BIG-IP requires that the BIG-IP admin install a RPM file that is released by Microsoft. However, recent releases of this RPM file are unable to be installed on BIG-IP due to an old version of rpm used on BIG-IP. It is possible to use a newer version of software to repackage the the .rpm file so that it can be installed on BIG-IP.

### Problem statement
Long story short: a BIG-IP admin seeking to install this RPM on BIG-IP will get this error:

{% highlight bash %}
curl -LOJ https://github.com/microsoft/AzureManagedHsmTLSOffload/releases/download/v1.1.0.02829/mhsm-pkcs11-1.1.0.02829-1.cm2.x86_64.rpm
 
rpm -ivh mhsm-pkcs11-1.1.0.02829-1.cm2.x86_64.rpm

Then we get an error message:
error: Failed dependencies:
        rpmlib(PayloadIsZstd) <= 5.4.18-1 is needed by mhsm-pkcs11-1.1.0.02829-1.cm2.x86_64

{% endhighlight %}

### Initial notes 

-	BIG-IP (latest version is 17.5) runs on *CentOS Linux release 7.3.1611 (Core)*. You can run `cat /etc/centos-release` to see this.
-	CentOS 7 is no longer supported by MS for this package. [README](https://github.com/microsoft/AzureManagedHsmTLSOffload?tab=readme-ov-file)
-	Release notes indicate the last CentOS 7 supported package is [v1.1.0.02802](https://github.com/microsoft/AzureManagedHsmTLSOffload/releases/tag/v1.1.0.02802). This was released Sept 2024, currently 2nd newest release.
  - It is somewhat unclear if the following version, v1.1.0.02829, supports CentOS 7. It may also.

### Workaround
After researching and asking ChatGPT, we learned we can do the following:
1. Build a new RHEL 9.4 server in Azure and ran these commands:

```bash
curl -LOJ https://github.com/microsoft/AzureManagedHsmTLSOffload/releases/download/v1.1.0.02802/mhsm-pkcs11-1.1.0.02802-1.cm2.x86_64.rpm

mkdir repack_rpm
cd repack_rpm

# Extract the RPM contents
rpm2cpio ../mhsm-pkcs11-1.1.0.02802-1.cm2.x86_64.rpm | cpio -idmv

#install fpm, which we will use to rebuild this package with gzip instead of Zstandard
sudo yum install -y ruby rubygems
gem install --no-document fpm

sudo yum install -y rpm-build

# Rebuild the RPM using gzip instead of Zstd
fpm -s dir -t rpm -n mhsm-pkcs11 -v 1.1.0.02802 --iteration 1.cm2 --architecture x86_64 --rpm-compression=gzip *
```

2.	Copy the new rpm file you just built to the BIG-IP.
3.	Install it: `rpm -ivh mhsm-pkcs11-1.1.0.02802-1.cm2.x86_64.rpm`

### Conclusion
The problem is that the version of `rpm` on BIG-IP is too old for RPM files that are packaged with Zstandard, a newer compression method than gzip. 