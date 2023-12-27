---
layout: single
toc: true
title:  "Centos and Red Hat notes, part 2"
categories: [redhat]
tags: [redhat, centos, linux]
excerpt: "Unlike Windows Server, which I learned 'from the ground up' by reading textbooks, I've just sort of learned Linux on the job over the years. Now I'm going back to learn the basics: history, fundamentals, and things I've always glossed over." #this is a custom variable meant for a short description to be displayed on home page
---
![Red Hat logo](/assets/red-hat-logo.svg){:style="display:block; margin-left:auto; margin-right:auto"} 
<hr />
[Part 1]({% post_url 2023-12-19-rhcsa-notes-1 %})<br/>
[Part 2]({% post_url 2023-12-26-rhcsa-notes-2 %})<br/>
<hr />
<!-- begin_excerpt -->
I've decided to take the exams for Red Hat Certified Systems Administrator (RHCSA) and Red Hat Certified Engineer (RHCE). I've been meaning to do this for 10 or 15 years! This post is just notes for myself. 

<!-- end_excerpt -->
**I usually use Ubuntu so these notes are specifically for learning CentOS.**
<br/><br/>
<hr />

### Disks, volumes, filesystems
- The proper command to list all devices we can use is in order ```pvs```, ```vgs``` and ```lvs```. It shows all **physical storages** and devices, **volume groups** and **logical volumes**.
- The **XFS** filesystem is the default for RHEL. It doesn't allow down-sizing, only extending.
- The command to extend a volume *and the filesystem on it* would be:
  ````bash
  # notice -r flag which indicates not only to resize logical volume but also filesystem on it
  lvextend –size 200M -r /dev/VOLUME_GROUP/LOGICAL_VOLUME
  ````
- **volume labels**. In order to give a label to a logical volume it must be unmounted, labeled, and mounted again:
  ````bash
  umount /LINK/TO/FILESYSTEM/MOUNT/POINT
  xfs_admin -L "myFS" /dev/VOLUME_GROUP/LOGICAL_VOLUME
  mount /LINK/TO/FILESYSTEM/MOUNT/POINT
  ````
- the **semanage** command allows you to interact with SELinux **contexts**. 
  - To work out how it's provided via yum, run ```yum whatprovides */semanage```
  - for my CentOS machine, the package is **policycoreutils-python-2.5-34.el7.x86_64**
  - so to install I will run ```yum install policycoreutils-python-2.5-34.el7.x86_64```
- to get SELinux context for a directory, the **-Z flag** can be used
  ```ls -Z /home```
  - the output on my machine is
    ```[centos@localhost ~]$ ls -Z /home
    drwx------. centos centos unconfined_u:object_r:user_home_dir_t:s0 centos
    drwx------+ john   john   unconfined_u:object_r:user_home_dir_t:s0 john
    ```
  - the part ending in _t is the SELinux context. So to apply this context to a different directory we could run
    ````bash
    semanage fcontext -a -t user_home_dir_t "/xfs(/.*)?"
    # However above command only assigns this **context** to the **policy**. In order to write it to the filesystem we need to invoke:
    restorecon -R /xfs
    ````
### Special permissions: SGID, SUID, and Sticky Bit
- Good reference: [https://www.redhat.com/sysadmin/suid-sgid-sticky-bit](https://www.redhat.com/sysadmin/suid-sgid-sticky-bit)
- Another reference: [https://www.scaler.com/topics/special-permissions-in-linux/](https://www.scaler.com/topics/special-permissions-in-linux/)
- **SUID**
  - **user + s(pecial)** is commonly noted as SUID. A file with SUID set *always executes as the user who owns the file*, regardless of the user passing the command. If the file owner doesn't have execute permissions, then use an uppercase S here.
  ````bash
  [centos@localhost ~]$ ls -l /usr/bin/passwd
  -rwsr-xr-x. 1 root root 27856 Mar 31  2020 /usr/bin/passwd
  ````
  Notice the **s** where the **x** usually is for the user permissions. This is a real-world example of when the SUID is useful.
- **SGID**
  - **group + s(pecial)** is commonly noted as SGID. 
    - If set on a file, it allows the file to be executed as the group that owns the file (similar to SUID)
    - If set on a directory, any files created in the directory will have their group ownership set to that of the directory owner
  - Example
    ```bash
    [tcarrigan@server article_submissions]$ ls -l 
    total 0
    drwxrws---. 2 tcarrigan tcarrigan  69 Apr  7 11:31 my_articles
    ```
    It is useful for directories that are often used in collaborative efforts between members of a group. Any member of the group can access any new file. 
- **Sticky Bit**
  - **other + t (sticky)** is a bit that does not affect individual files. However at the directory level, it restricts file deletion. Only the **_owner_ of the directory** (and root) can delete files within that directory.
  - A common example is the ```/tmp``` directory.
    ```bash
    [centos@localhost ~]$ ls -ld /tmp
    drwxrwxrwt. 9 root root 4096 Dec 26 03:14 /tmp
    ```
    You can see the **t** where the **x** normally is for the "other" users. 
- **Setting special permissions**
  - Using the numerical method, we need to pass a fourth, preceding digit in our ```chmod``` command. The digit used is calculated similarly to the standard permission digits:
    - Start at 0
    - SUID = 4
    - SGID = 2
    - Sticky = 1

    Here is an example where we want the SGID bit set on the directory, so we append **2** before the permissions and run ```chmod 2770``` to set SGID.
    ```bash
    [tcarrigan@server article_submissions]$ chmod 2770 community_content/
    [tcarrigan@server article_submissions]$ ls -ld community_content/
    drwxrws---. 2 tcarrigan tcarrigan 113 Apr  7 11:32 community_content/
    ```

### Partition labels and file system names
- Good reference: [https://superuser.com/questions/1099232/what-is-the-difference-between-a-partition-name-and-a-partition-label/1099292](https://superuser.com/questions/1099232/what-is-the-difference-between-a-partition-name-and-a-partition-label/1099292){:target="_blank"}

### Linux Access Control Lists:
- Good reference [https://www.redhat.com/sysadmin/linux-access-control-lists](https://www.redhat.com/sysadmin/linux-access-control-lists)
- **getfacl** will get existing ACL's on a directory or file.
- **setfacl** will set a ACL on a directory or file.

### Virtual Console
```grubby –update-kernel=ALL –args="console=ttyS0"```

### Cron
- Good reference: [https://www.redhat.com/sysadmin/automate-linux-tasks-cron](https://www.redhat.com/sysadmin/automate-linux-tasks-cron)
- Another: [https://www.redhat.com/sysadmin/linux-cron-command](https://www.redhat.com/sysadmin/linux-cron-command)
- ```crontab -l``` will list the existing crontab for the current user
- ```crontab -e``` will edit it
- the ```-u (user)``` will allow you to see the crontab for other users. Eg: ```sudo crontab -u root -l```
- there is only one file per user on the system. There is no extension to worry about in the file. Just edit it with vi and this is the format:
![crontab format](/assets/red-hat-notes/crontab.jpg)


### Gather system statistics daily at 11pm
- The command to gather statistics is called ```sar``` and to install it we will do:
  ```bash
  yum install sysstat
  systemctl enable sysstat
  systemctl start sysstat
  ```
- This will add it to the crontab of the root user
  ```bash
  su crontab -u root -e
  # make the necessary edits
  0 23 * * * /usr/bin/sar -A > /var/log/consumption.log
  ```
  With the above, the command will run at 11pm every day in the context of the root user.

### Linux systemd targets
- Good reference: [https://opensource.com/article/20/5/systemd-startup](https://opensource.com/article/20/5/systemd-startup)
- Here is the list of all targets in systemd:<br>
<br>
  0 poweroff.target<br>
  1 rescue.target<br>
  2 multi-user.target<br>
  3 multi-user.target<br>
  4 multi-user.target<br>
  5 graphical.target<br>
  6 reboot.target<br>
- This will set the default target to boot into
  ```bash
  systemctl set-default graphical.target
  ```
- You can check with this command
  ```bash
  systemctl get-default
  ```