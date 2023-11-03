---
layout: post
title:  "How I deploy OpenShift in a hurry"
date:   2023-11-01 09:00:00 -0400
categories: openshift
---

Like many posts, this one is going to be quick and rough. 
- Today is Nov 2, 2023, and the [latest release of OpenShift](https://access.redhat.com/support/policy/updates/openshift#dates){:target="_blank"} is cuurently version 4.14.
- If you want a specific version of OpenShift, [click here](https://access.redhat.com/solutions/5149581){:target="_blank"}. 

### My previous work with Openshift

I have needed to dive into Red Hat OpenShift now and then in the course of my work at F5. We're a great partner of Red Hat and we have many mutual customers.

My most public-facing work so far has been the work I did to automate this architecture when I wrote a nice article about integrating F5 BIG-IP into [Azure Red Hat Openshift](https://azure.microsoft.com/en-us/products/openshift){:target="_blank"} (ARO).

Article: [Running F5 with managed Azure RedHat OpenShift](https://community.f5.com/t5/technical-articles/running-f5-with-managed-azure-redhat-openshift/ta-p/291157){:target="_blank"}

![image 'F5 BIG-IP and Azure RedHat Openshift'](/assets/f5-azure-redhat-openshift.png)

### How I prefer to deploy OpenShift in a hurry
I like a cheaper option than ARO for a lab or demo, and also something that I can power down easily to reduce cost. Here's instructions for quickly deploying a **self-managed** OpenShift cluster in AWS. 

#### Pre-requisites

1. Create a VPC in AWS<br/>
You need at least 4x subnets in 2x Availability Zones.
- 2x external subnets with a default route out Internet-Gateway (must be public-facing)
- 2x internal subnets (don't need to be Internet-facing). 

2. [Register for an account](https://www.redhat.com/en/technologies/cloud-computing/openshift) with Red Hat and get access to a [pull secret](https://cloud.redhat.com/openshift/install/pull-secret).

3. Prepare a file called ```install-config.yaml```. I have an example [here](/assets/install-config.yaml){:target="_blank"}. You can see I have pre-filled most values, but you will need to edit at least the following fields: **baseDomain**, **platform>aws>subnets**, **pullSecret**, and **sshKey**.

{% highlight yaml %}
baseDomain: my-f5.com #you will need a domain that is hosted in AWS Route53 here.
platform:
  aws:
    region: us-east-1 # the AWS region that your VPC is in
    subnets:
    - subnet-xxxxxxxxxxxxxxxxx # your public subnet id in AZ1
    - subnet-xxxxxxxxxxxxxxxxx # your public subnet id in AZ2
    - subnet-xxxxxxxxxxxxxxxxx # your private subnet id in AZ1
    - subnet-xxxxxxxxxxxxxxxxx # your private subnet id in AZ2
publish: External
pullSecret: 'my-pull-secret' #get your pull secret from https://cloud.redhat.com/openshift/install/pull-secret
sshKey: |
  ssh-rsa xxx...yyy imported-openssh-key #use your own key here to allow you to access the ec2 instances created with ssh.
{% endhighlight %}

{:start="4"}
3. Download some binaries from Red Hat here: [https://cloud.redhat.com/openshift/install/aws/user-provisioned](https://cloud.redhat.com/openshift/install/aws/user-provisioned). <br/>You can choose to get the installer for Mac OS or Linux, and the CLI can be downloaded for Mac, Linux, or Windows.
* Download the installer (UPI – User provisioned Infrastructure)
* Also, you'll need to download the client CLI from the same URL above. This gets installed on your laptop and you use it to interact with Openshift.

#### Installation directions
1. Follow these instructions for installing OpenShift onto existing VPC in AWS: [https://docs.openshift.com/container-platform/4.14/installing/installing_aws/installing-aws-vpc.html#installation-obtaining-installer_installing-aws-vpc](https://docs.openshift.com/container-platform/4.14/installing/installing_aws/installing-aws-vpc.html#installation-obtaining-installer_installing-aws-vpc)
* uncompress the downloaded files
* you will need to create a config file called ```install-config.yaml``` that will contain details of your deployment. In this example the most important detail to be aware of is the subnet Ids.
  * I have an example ```install-config.yaml``` file for you [here](/assets/install-config.yaml).
* make sure you create a new directory for every OpenShift cluster you create. In my example, my directory is called ```install_ocp```

{% highlight bash %}
tar -xzvf openshift-install-linux.tar.gz
tar -xzvf openshift-client-linux.tar.gz
mkdir install_ocp
cp install-config.yaml-backup install_ocp/install-config.yaml
./openshift-install create cluster --dir=install_ocp/ --log-level=info
{% endhighlight %}

**Important**: Do not delete the installation program or the files that the installation program creates. Both are required to delete the cluster.

#### Access the cluster
- Once your cluster build is complete you will see in the logs that your User Interface is ready.
  - the default username is ```kubeadmin``` and the password will be in the logs.
- Use the ```oc``` utility or the ```kubectl``` command line to authenticate and administer your cluster.

#### Destroy the cluster

Instructions for destroying the cluster are [here](https://docs.openshift.com/container-platform/4.14/installing/installing_aws/uninstalling-cluster-aws.html).

This will not destroy the AWS infrastructure resources (VPC, Route53 domain, etc) so you can clean those up manually.

{% highlight bash %}
./openshift-install destroy cluster --dir=install_ocp/ --log-level=info
{% endhighlight %}