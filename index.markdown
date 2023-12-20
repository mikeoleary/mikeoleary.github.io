---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: splash
#layout: home-custom
title: Engineering together
header:
  overlay_color: "#000"
  overlay_filter: "0.5"
  overlay_image: /assets/splash-page/header-image-engineering-1600x524.jpg
  #actions:
  #  - label: "Download"
  #    url: "https://github.com/mmistakes/minimal-mistakes/"
  #caption: "Photo credit: [**Pexels**](https://www.pexels.com/photo/prosthetic-arm-on-blue-background-3913025/)"
excerpt: "Tech blog on Kubernetes, cloud architecture, and automation."
intro:
  - excerpt: >-
     Welcome to my tech blog.<br/>
     <br/>
     The purpose here is to **take notes** and **share learning**. Most of this blog will revolve around cloud architecture, Kubernetes, automation, and related technologies.<br/>
     <br/>
feature_row:
  - image_path: assets/splash-page/tech-blog.jpg
    alt: "All posts"
    title: "All posts"
    excerpt: "Kubernetes, cloud, automation, and other professional topics"
    url: "/all-posts/"
    btn_label: "Read More"
    btn_class: "btn--primary"
  - image_path: /assets/splash-page/community-hands.jpg
    #image_caption: "Image courtesy of [Unsplash](https://unsplash.com/)"
    alt: "Community"
    title: "Community"
    excerpt: "Boston Kubernetes Meetup, CNCF Boston chapter, and other community activities"
    url: "/meetup/"
    btn_label: "Read More"
    btn_class: "btn--primary"
  - image_path: /assets/splash-page/about-this-site.jpg
    title: "About"
    excerpt: "About this site"
    url: "/about/"
    btn_label: "Read More"
    btn_class: "btn--primary"
---
{% include feature_row id="intro" type="center" %}

{% include feature_row  %}