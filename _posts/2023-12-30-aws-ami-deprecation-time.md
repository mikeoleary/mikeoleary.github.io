---
layout: single
title:  "Terraform, AWS AMI and the DeprecationTime attribute"
categories: [aws]
tags: [aws, terraform]
excerpt: "Today I learned the hard way about AWS AMI deprecation times. Here's some quick notes that may save you some time one day." #this is a custom variable meant for a short description to be displayed on home page
---
This post covers [deprecated AMIs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ami-deprecate.html) in case you are not familiar with that.

I'm running through a lab that I plan to present at [F5 AppWorld 2024](https://www.f5.com/appworld). The lab uses Terraform to deploy F5 BIG-IP VE's, among other things.

*Things worked fine when I tested this lab a few weeks ago.* Now my automation is failing!
### The problem
Here's an excerpt of the Terraform configuration that is now failing:

My terraform.tfvars file:
{% highlight json %}
f5_ami_search_name  = "F5 BIGIP-17.1.*PAYG-Adv WAF Plus 25Mbps*"
{% endhighlight %}

My ami-search.tf file:
{% highlight json %}
data "aws_ami" "bigip" {
  most_recent = true
  filter {
    name   = "name"
    values = [var.f5_ami_search_name]
  }
  owners = ["aws-marketplace"]
}
{% endhighlight %}

However, when I run ```terraform plan``` I get the error message:
```
│ Error: Your query returned no results. Please change your search criteria and try again.
│
│   with data.aws_ami.bigip,
│   on ami-search.tf line 3, in data "aws_ami" "bigip":
│    3: data "aws_ami" "bigip" {
```

### Why is Terraform not finding the AMI?
I know that an AMI (```ami-056a053acf172f5b8```) does exist that matches this search criteria. It has a **name** attribute of ```F5 BIGIP-17.1.0.1-0.0.4 PAYG-Adv WAF Plus 25Mbps-230407095221-3c272b55-0405-4478-a772-d0402ccf13f9```

I can find it with the AWS CLI: 

{% highlight json %}
ubuntu@ubuntu-Virtual-Machine:~$ aws ec2 describe-images --region us-west-2 --image-ids ami-056a053acf172f5b8
{
    "Images": [
        {
            "Architecture": "x86_64",
            "CreationDate": "2023-04-21T23:37:51.000Z",
            "ImageId": "ami-056a053acf172f5b8",
            "ImageLocation": "aws-marketplace/F5 BIGIP-17.1.0.1-0.0.4 PAYG-Adv WAF Plus 25Mbps-230407095221-3c272b55-0405-4478-a772-d0402ccf13f9",
            "ImageType": "machine",
            "Public": true,
            "OwnerId": "679593333241",
            "PlatformDetails": "Linux/UNIX",
            "UsageOperation": "RunInstances",
            "ProductCodes": [
                {
                    "ProductCodeId": "3k7bic6nm4bveoy25v1kxvvuh",
                    "ProductCodeType": "marketplace"
                }
            ],
            "State": "available",
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/xvda",
                    "Ebs": {
                        "DeleteOnTermination": true,
                        "SnapshotId": "snap-05f0661b9a1316786",
                        "VolumeSize": 82,
                        "VolumeType": "gp2",
                        "Encrypted": false
                    }
                }
            ],
            "Description": "F5 BIGIP-17.1.0.1-0.0.4 PAYG-Adv WAF Plus 25Mbps-230407095221",
            "EnaSupport": true,
            "Hypervisor": "xen",
            "ImageOwnerAlias": "aws-marketplace",
            "Name": "F5 BIGIP-17.1.0.1-0.0.4 PAYG-Adv WAF Plus 25Mbps-230407095221-3c272b55-0405-4478-a772-d0402ccf13f9",
            "RootDeviceName": "/dev/xvda",
            "RootDeviceType": "ebs",
            "SriovNetSupport": "simple",
            "VirtualizationType": "hvm",
            "DeprecationTime": "2023-11-15T14:44:00.000Z"
        }
    ]
}
{% endhighlight %}

### Searching for deprecated AMI's
I tried a few things, including searching for the AMI ID specifically in Terraform like this, to no avail:

{% highlight json %}
data "aws_ami" "bigip" {
  most_recent = true
    filter {
    name   = "image-id"
    values = ["ami-056a053acf172f5b8"]
  }
  owners = ["aws-marketplace"]
}
{% endhighlight %}

Then I realized that the ```DeprecationTime``` attribute was now in the past. I quickly found the Terraform [documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami#include_deprecated) and learned I could add ```include_deprecated = true``` to have deprecated AMIs found by Terraform.

Now, this Terraform code is doing what I expect (see line 7 below):

```
data "aws_ami" "bigip" {
  most_recent = true
  filter {
    name   = "name"
    values = [var.f5_ami_search_name]
  }
  include_deprecated = true
  owners = ["aws-marketplace"]
}
```
