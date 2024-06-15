---
layout: single
title:  "New laptop set up details and tricks"
categories: [windows]
tags: [windows]
excerpt: "My laptop SSD died and I've been issued a replacement. This documents my workstation set up in case I need to do it again." #this is a custom variable meant for a short description to be displayed on home page
gallery:
  - image_path: /assets/new-laptop/backyard-1.jpg
    url: /assets/new-laptop/backyard-1.jpg
    title: "Rear deck"
  - image_path: /assets/new-laptop/backyard-2.jpg
    url: /assets/new-laptop/backyard-2.jpg
    title: "Outdoor furniture cushions"
toc: true
---
<figure>
    <a href="/assets/new-laptop/new-laptop-setup.jpg"><img src="/assets/new-laptop/new-laptop-setup.jpg"></a>
    <figcaption>My new laptop. For cleaner looking presentations, I never save any files to my desktop.</figcaption>
</figure>

#### Background
It's funny how unrelated event seem to happen together. Recently I had to rebuild the Ubuntu Linux VM that I use for everyday tasks, so I [documented the setup]({% post_url 2024-06-07-ubuntu-server-post-deploy-script %}) to save time in future. 

A couple days later, my trusty Windows10 Dell laptop has died. The SSD disk is toast, and I certainly don't want to invest time into trying to salvage this desktop hardware. It's probably 3-4 yrs old and out of warranty. Some engineers enjoy tinkering with desktop hardware. These days I just don't have the time.

#### New laptop
Funny story. My laptop died on a Tue. I called our helpdesk. I figured I had until at least Thur or Fri before F5 could express ship me a replacement (F5 is HQ'd in Seattle and I live in Boston). I planned to get some yardwork done while working from my cell phone where possible. I went outside and pressure-washed my deck, outdoor furniture cushions, and anything else I could find in my back yard.

{% include gallery id="gallery" caption="When your laptop dies you have limited time to get other stuff done during daytime hours!"  %}

It turns out that a Service Desk employee lives 15 mins from my house and was able to deliver a new laptop, pre-built with the corporate image, right to my front door. I was without a laptop for less than a day. Lucky me...

<figure>
    <a href="/assets/new-laptop/workstation-notebook-precision-16-5680-nt-black-gallery-4.jpg"><img src="/assets/new-laptop/workstation-notebook-precision-16-5680-nt-black-gallery-4.jpg"></a>
    <figcaption>Dell Precision 5680.</figcaption>
</figure>

#### Best practices I'm documenting for my local workstation
I prefer to use a Windows laptop because it's the least hassle for the corporate worker like myself. I know most engineers love Mac or Linux workstations. I've chosen to pick other battles and am not embarrassed to say that I find Windows easy to use. I was using Windows 10, but this new laptop runs Windows 11. No problem.

I store almost everything in "the cloud" that I can. 
- Chrome bookmarks and settings are saved in my Google profile. These carried over to the new machine, of course.
- All of my documents are in OneDrive. This is managed by the enterprise and I'm happy to leave it that way.
- **Nothing** is saved to my desktop. I hate to see people get muddled by scattered files saved in various local folders or, worse, the desktop where others see the files if they share their monitor in a Zoom call.
- My local Downloads folder acts like a temp directory for me. Most recent files exist there, but I don't rely on anything persisting there. 

A few other tips that I like to follow.
- Often, less is more.
  - I don't use the iPad you see pictured above (it's there for unrelated reasons in the pic).
  - I don't use a virtual whiteboard to draw diagrams live in front of customers. I find that distracting.
  - I have a fancy camera and a basic mic, but no fancy software. 
- I like to offload to hardware to keep things simple.
  - The ATEM Mini pictured offloads the need to run a driver to connect my camera as a webcam. It can do fancy things. I use it simply to switch cameras, nothing else.
- My office is pretty clean. You'll never see a Zoom background with loose papers or half-finished projects from me. No conversation starters like Lego models or Star Wars posters. That's just not me, and I'd rather focus my meetings on solving the problems at hand. Not every body needs to hear about my hobbies.
- Lastly, I'll say that I had Ethernet cable run to my office when I moved into the house.

#### Setting up my new laptop
Any other software or configuration that was local to my old laptop was so minor that re-installation has not been a problem. Still I will document what I've done to set up a new Windows 11 laptop, because inevitably I'll need to remember this in future.

````
- Notepad++
- VS Code
 - remote SSH extension from Microsoft
- Pagent
  - shortcut to launch this at startup and load a priv key: https://community.sw.siemens.com/s/article/How-to-setup-Pageant-to-run-at-Startup-and-load-your-private-key-automatically
- putty
- puttygen
  - load this at startup. I seem to use it all the time.
- Hyper V
- WinSCP
- Blackmagic ATEM switcher software
- Elgato keylight software called Control Center
````

That's it! A few hours of downtime but now I'm back running, with almost zero learning curve for the new Windows 11 platform. 

