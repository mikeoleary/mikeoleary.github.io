---
layout: single
title:  "FTP server on K8s with F5"
categories: [kubernetes]
tags: [kubernetes]
excerpt: "FTP is a legacy protocol that requires specific knowledge. Most young engineers have never used it. Kubernetes is modern, but complicated. Most senior engineers have never used it. Let's do this!" #this is a custom variable meant for a short description to be displayed on home page
---

### Background
After 5 years honing Kubernetes expertise, I was happy to undertake a challenge: expose an FTP server from within Kubernetes, in the public cloud, using F5 BIG-IP. Here's how I did this.

### Isn't FTP a legacy technology?
Yes, FTP has been around since the early 70's. It was designed for efficient tranfer of files, and although it's insecure by default, it's still commonly seen today.

#### Advantages of FTP
As opposed to other file transfer methods, FTP does offer a few advantages:
- unlike uploading files over other protocols, like HTTP, you can resume file transfers if a connection is lost
- allows for a queue of files to be uploadeded or downloaded
- faster / more efficient than HTTP
- no file size limitations
- extremely common platform offered widely for many years

#### Disadvantages of FTP
FTP is considered legacy because of a few limitations. There's workaround and enhancements, so I'll give a basic overview:
- insecure by default
  - Standard FTP sends username, password, and files in clear text
  - Servers can be spoofed to send data to the wrong computer
- complexity
  - by default, the control channel is over TCP/21, but is usually configurable
  - by default, the data channel is over a random high port, if using Passive mode, and is usually configurable
  - by default, the data channel is over TCP/20, if using Active mode, and is usually configurable
- active vs passive
  - active mode requires the server to establish a connection to a client. This is an outdated model disallowed by most firewalls
  - passive mode requires outbound connections over random high ports, which are also disallowed by most firewalls. Therefore, firewalls or L4 network devices (like BIG-IP) must be "ftp aware"

There are many more complex advantages and disadvantages, but to summarize: running and securing FTP servers requires knowledge of the protocols, not just general network and security knowledge.

#### Enhancements and common FTP servers
FTP is commonly seen, but it's very common to see enterprises run a server that also offers FTPS and SFTP. While they sound similar, these are different protocols. FTPS allows for encryption of the control channel and (optionally) the data channel. SFTP adds file transfers upon the SSH protocol, meaning all transfers can happen over a single TCP connection on port 22. However, SFTP is difficult to proxy, and FTPS requires additional knowledge on top of FTP.

For this reason, enterprise-level FTP servers are usually well built out with a customer support team, static IP addresses, commercial software and support. Changes are usually slow, upgrades infrequent, and transfers frequent, large, and critical. For example, large financial customers may have longstanding practices built around FTP, and so require very high confidence in the redundancy, security, and support of their FTP systems.

My recent customer is 

### Why run FTP on Kubernetes?

### How to run your FTP server on Kubernetes and still get enterprise-level protection

