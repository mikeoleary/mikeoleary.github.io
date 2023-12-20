---
layout: single
title:  "Centos and Red Hat notes, part 1"
categories: [redhat]
tags: [redhat, centos, linux]
excerpt: "Unlike Windows Server, which I learned 'from the ground up' by reading textbooks, I've just sort of learned Linux on the job over the years. Now I'm going back to learn the basics: history, fundamentals, and things I've always glossed over." #this is a custom variable meant for a short description to be displayed on home page
---
![Red Hat logo](/assets/red-hat-logo.svg){:style="display:block; margin-left:auto; margin-right:auto"} 
<hr />
<br/>
<!-- begin_excerpt -->
I've decided to take the exams for Red Hat Certified Systems Administrator (RHCSA) and Red Hat Certified Engineer (RHCE). I've been meaning to do this for 10 or 15 years! This post is just notes for myself. 

<!-- end_excerpt -->
**I usually use Ubuntu so these notes are specifically for learning CentOS.**
<br/><br/>
<hr />

### Users, passwords, etc.
- **create user**. This will create a user named **john** with a uid/gid of 2000
  - ```sudo useradd -u 2000 john```
  - ```sudo passwd john``` #enter password for john here
- **set password validity for user**. The ```chage``` command fordes users to change passwords to comply with password-aging policies
  - ```sudo chage -l john``` #the -l flag will list the password expiry date, date of last password set, and other info for a user
  - ```sudo chage -E $(date -d +30days +%Y-%m-%d)``` # the -E flag will set a date to expire their password
- **change password**. ```passwd```
- **change root password**. ```sudo passwd root```
- **change root password when you don't have sudo privilege**. You need physical access to machine.
  - reboot machine and press *e* during GRUB loader screen. That will open an editor with current kernel boot options.
  - find line starting with **linux16** and add **rd.break** at the end and hit Ctrl+X to exit<br/>
    ```mount -o remount,rw /sysroot/```<br/>
    ```chroot /sysroot```<br/>
    ```passwd``` to change your password<br/>
    ```touch /.autorelabel```<br/>
    ```exit```<br/>
    ```exit``` or ```reboot```<br/>

- **sudoers file**. Your user must be in this if you want to run ```sudo``` in front of your commands.
{% highlight bash %}
su - root                       #switch to the root user
#Option 1: add the user to the 'wheel' group
usermod -a -G wheel centos      #add the user 'centos' to the group 'wheel' which has the rights to run the sudo command
#Option 2: add the user directly to /etc/sudoers file
visudo
#add a line like this to the file 'centos ALL=(ALL:ALL) ALL'
{% endhighlight%}
- **change hostname**
  - ```nmtui``` or
  - edit ```/etc/hostname``` or
  - use the command ```hostnamectl set-hostname SOME_NAME``` or
  - use nmcli: ```nmcli general hostname SOME_NAME```<br/>
  **NB:** if you want to change the 'pretty hostname' without restarting the OS, you can use ```hostnamectl set-hostname SOME_NAME --pretty``` or create/edit the file ```/etc/machine-info``` so that it has the pretty hostname in it.

### Networking, routing, etc
- **My Hyper-V on Win10**. When I deployed Centos 7.9.2009 on Hyper-V using [the ISO found here](https://mirrors.mit.edu/centos/7.9.2009/isos/x86_64/), I typed ```nmcli``` and it showed the interface ```eth0``` was in a disconnected state. Using the console UI in Hyper-V, I typed ```nmcli device connect eth0``` and then the interface obtained a DHCP address immediately.

- **Set a static IP**. Edit ```/etc/sysconfig/network-scripts/ifcfg-eth0```. I'm experienced enough not to type everything out, but in my case I'll note that ```BOOTPROTO``` can be changed from **dhcp** to **static** and that ```ONBOOT``` can be changed to **yes**. My file now looks like this:

{% highlight bash %}
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static        #edited from 'dhcp' to 'static'
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eth0
UUID=3abc8a88-85fe-4a80-8e29-ed1ff1dcf739
DEVICE=eth0
ONBOOT=yes              #edited from 'no' to 'yes'
IPADDR=172.23.110.145   #I added this line
NETMASK=255.255.240.0   #I added this line
GATEWAY=172.23.96.1     #I added this line
DNS1=8.8.8.8            #I added this line
DNS2=8.8.4.4            #I added this line
{% endhighlight %}

- On Centos, use the ```ip route``` command (not just ```route``` like Ubuntu)
- **Add a secondary IP address.**<br/>
  Option 1: ```nmtui``` and follow the interface to add a secondary IP. Run ```service network restart``` after that.<br/>
  <br/>
  Option 2: Edit the config files directly. Can only be done if ```NM_CONTROLLED="no"``` or is not configured at all on the interface. You would just add these lines to the above config file:
{% highlight bash %}
IPADDR1=172.23.110.146
PREFIX1=20
NETMASK1=255.255.240.0
{% endhighlight %}
- **Add a temporary secondary IP address**. This will last only until server reboot our the next network service restart.
{% highlight bash %}
ip a add 172.23.110.146/20 dev eth0
{% endhighlight %}

- restart network service: ```service network restart```

### SE Linux
- **install SE Linux**. This worked for me on Centos 7.9
{% highlight bash %}
sudo yum install policycoreutils policycoreutils-python setools setools-console setroubleshoot
{% endhighlight %}
- **disabled vs enforcing**<br/>
The file ```/etc/selinux/config``` should have ```SELINUX=enforcing``` or ```permissive``` or ```disabled``` (editing this file and rebooting is the way to ensure that the mode will persist after reboot)
- ```getenforce``` - this command will tell you which mode you are in
- ```sudo etenforce 0``` for permissive or ```sudo setenforce 1``` for enforcing (does not persist after reboot)
- ```sestatus``` to check status of SE Linux

### Apache
- ```sudo yum install -y httpd``` - this will install Apache
- remember to allow service to interact with network through firewall
{% highlight bash %}
#notice the '--permament' option (in order to save rule to survive during reboots)
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
{% endhighlight %}
- remember to start a service after installing it, and to enable it (to autostart after reboot)
{% highlight bash %}
systemctl enable httpd
systemctl start httpd
{% endhighlight %}