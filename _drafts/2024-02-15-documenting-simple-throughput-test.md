---
layout: single
title:  "Simple BIG-IP throughput testing in Azure"
categories: [azure, aws]
tags: [azure, aws]
excerpt: "This is a basic outline of my current testing of throughput for cloud-based appliances that I deal with at work. " #this is a custom variable meant for a short description to be displayed on home page
---

This is a basic outline of my current testing of throughput for cloud-based appliances that I deal with. I love F5 technology and watching it work in cloud, so this is an attempt to document a testing scenario. 

![BIG-IP testing](/assets/throughput-testing-in-cloud/testing-architecture1.png)

### BIG-IP througput testing
I am testing the throughput of F5 BIG-IP virtual machines, different drivers, and different configurations. 

In the diagram above I have 2 BIG-IP VM's, one of which is a default marketplace image (a BYOL image), and another that is my experimental image. This way I have a baseline for testing and a candidate for configuration and tweaking.

You probably should not test this without advice/help from an F5 engineer. I'm using an evaluation license to take advantage of additional cores, as well as talking to internal resources with specialized knowledge.

#### Device configurations

|iperf||
|---|---|
|OS|Ubuntu 22.04|
|VM size|Standard_D64ds_v4|
|Accelerated Networking|Enabled|

|F5 BIG-IP||
|---|---|
|Version|17.1.1.1 build 0.0.2|
|VM size|Standard_D64s_v4|
|VS type|Performance L4|
|Accelerated Networking|Disabled|
|License|(Specially obtained)|

#### Results
- With the standard iperf test running host->host, I see beween 30 and 45 Gbps in testing with this command: 
```
iperf -c 10.0.1.4 -w 8m -P 64
```
- When traversing BIG-IP with no tuning (default driver, default queue size, Perf L4 VIP, etc) I get about 5 Gbps. This is now my baseline and I have several variables to tweak. Updates to come on this. I plan to test drivers, queue length, VS type, and maybe TCP profile settings on BIG-IP.

#### Interesting notes so far
- the `-P` for parallelism makes a difference because a single tcp stream throughput does not seem to achieve as much as multiple streams. 
  - I've been using `-P 64` but I seem to get about the same thoughput with `-P 10` and `-P 128`. 
- the `-t` for duration does not seem to affect throughput, so I'm leaving it off (default is 10 seconds)
- the `-w` for tcp window size is very meaningful. I've been using `-w 8m` to request 8MB tcp window size, but I also had to tweak `/etc/sysctl.conf` following [this article](https://netbeez.net/blog/tcp-window-size/) to allow it.

#### Handy links worth noting:
- https://fasterdata.es.net/performance-testing/network-troubleshooting-tools/iperf/multi-stream-iperf3/
- https://netbeez.net/blog/tcp-window-size/





