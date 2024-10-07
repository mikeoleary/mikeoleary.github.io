---
layout: single
title:  "Quickly test UDP on XC with NTP"
categories: [f5]
tags: [f5]
excerpt: "I can never remember this one-liner so here it is to bookmark." #this is a custom variable meant for a short description to be displayed on home page
---
Sometimes I want to test UDP connectivity, usually through F5 XC. Here's a quick way to set up NTP using Ubuntu.

I basically copied [this link](https://www.tecmint.com/install-ntp-server-and-client-on-ubuntu/) for setting up NTP on client and server.

### NTP Server
- deployed Ubuntu 22.04 LTS

````bash
sudo apt update -y
sudo apt install ntp -y
sudo systemctl status ntp
````

### NTP Client

````bash
sudo apt update -y
sudo apt install ntpdate -y
````

### Test from client to server
````bash
ntpdate -q [ip-address-of-server]
````

### F5 XC as UDP proxy and load balancer
To proxy this through F5 XC, you cannot use HTTP or TCP Load Balancers, obviously. You must create the equivalent with the Virtual Host objects:

1. Create Endpoint (IP address of NTP server)
2. Create Cluster (group of endpoints)
3. Create Route (send traffic to cluster)
4. Create Advertise Policy (listen on a given IP address on a CE, or "Virtual Network" and "vesi-io-shared/public" if you want to advertise to public Internet)
5. Create Virtual Host object to link all of these together.

<figure>
    <a href="/assets/ntp-on-xc/xc-screenshot-udp-advertise-policy.png"><img src="/assets/ntp-on-xc/xc-screenshot-udp-advertise-policy.png"></a>
    <figcaption>Screenshot of Advertise Policy that will use default tenant IP on RE's</figcaption>
</figure>

#### Notes
- at this time it looks like you cannot have a custom VIP for internet-facing traffic for UDP traffic.
- at this time it looks like Performance and Application Dashboards do not include UDP traffic.

#### Windows
I am not 100% sure but I think I used [this tool](https://www.timesynctool.com/) to easily demo with a Windows desktop as the NTP client.