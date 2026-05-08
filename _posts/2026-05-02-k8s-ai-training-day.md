---
layout: single
title: "K8s and AI Training Day 2"
date: 2026-05-02
categories: [ai]
tags: [ai, kubernetes]
excerpt: "Summary of K8s and AI training day #2" #this is a custom variable meant for a short description to be displayed on home page
toc: true
gallery:
  - url: /assets/k8s-training-day-2/ben-speaking.JPEG
    image_path: /assets/k8s-training-day-2/ben-speaking.JPEG
    alt: "Ben Speaking"
    title: "Ben Speaking"
  - url: /assets/k8s-training-day-2/nina-speaking.JPEG
    image_path: /assets/k8s-training-day-2/nina-speaking.JPEG
    alt: "Nina Speaking"
    title: "Nina Speaking"
  - url: /assets/k8s-training-day-2/mike-at-back.JPEG
    image_path: /assets/k8s-training-day-2/mike-at-back.JPEG
    alt: "Class Shot"
    title: "Class Shot"
gallery2:
  - url: /assets/k8s-training-day-2/isabella-delish-lunch.JPEG
    image_path: /assets/k8s-training-day-2/isabella-delish-lunch.JPEG
    alt: "Delicious lunch Isabella!"
    title: "Delicious lunch Isabella!"
  - url: /assets/k8s-training-day-2/cluster-architecture.jpg
    image_path: /assets/k8s-training-day-2/cluster-architecture.jpg
    alt: "Cluster Architecture"
    title: "Cluster Architecture"
---

<figure>
    <a href="/assets/k8s-training-day-2/organizer-team.JPEG"><img src="/assets/k8s-training-day-2/organizer-team.JPEG"></a>
    <figcaption>Organizer team</figcaption>
</figure>

On Saturday, May 2, we held our second full-day Kubernetes training event with the Boston Kubernetes Meetup community at the Microsoft NERD Center in Cambridge. The focus this year was **AI networking on Kubernetes** — moving beyond foundational Kubernetes networking into the operational, security, and architectural challenges introduced by modern AI systems and AI agents.

The event was intentionally hands-on and instructor-led, with attendees spending the day working through labs, discussions, demos, and real-world design considerations. About 30 attendees gave up a Saturday to attend. They ranged from relative beginners with K8s to experts who knew K8s but came for the LLM API overview or the agentic governance talks.

A huge thank-you goes to everyone who attended, asked questions, and helped create a highly interactive atmosphere throughout the day.

---

# Why We Ran This Event

AI systems are changing traffic patterns, application architectures, and operational models inside Kubernetes environments. Traditional north/south web traffic assumptions no longer fully apply. Clearly, agentic AI is bringing even greater compliance and governance challenges to the already new threat landscape of Generative AI.

I believe everyone in IT needs to know at least the basics of:
- Large language models (LLMs)
- AI gateways
- AI agents
- Retrieval systems
- Vector databases
- Tool-calling frameworks
- External APIs
- Multi-model inference services

These systems introduce new networking and security concerns that many Kubernetes engineers have not had to solve before. The goal of the training day was to bridge that gap with practical content rather than high-level theory.

---

# Agenda Overview

The event agenda focused on both foundational and advanced topics related to AI traffic inside Kubernetes environments. Sessions included:

- K8s fundamentals deck (First presentation from Michael)
- Killercoda.com/f5-se
  - Hello App Basics (online lab showing pod-to-pod communication)
- LLM API & prompting best practices (Second presentation from Michael)
- Hands-on demo (presentation from Ben)
  - use a Github codespace
  - deploy kind, a cluster, a demo app
  - deploy k8gpt, grafana, and kagent
- agentgateway for agent governance (Nina's presentation)
  - Sandbox environment (lab we did with Nina)
  - agentgateway and MCP auth (additional lab we did not cover)

The structure combined whiteboarding, architecture walkthroughs, live demos, and hands-on labs so attendees could immediately apply concepts during the sessions themselves.

---

{% include gallery caption="Pics from Sat May 2, 2026" %}

---

# Instructor-Led and Community-Focused

The event was led by myself, Isabella Langan, Benjamin Hautefeuille, and Nina Polshakova. There was a very strong emphasis on practical engineering discussions rather than vendor marketing - this is something I hold very strictly to, especially for Training Days.

---

{% include gallery id="gallery2" caption="More pics from Sat May 2, 2026" %}

## Key Takeaways

### AI Changes Traffic Patterns

AI applications often create dramatically different internal traffic flows compared to traditional microservices applications. Tool-calling agents and chained inference systems can generate complex service-to-service communication paths that are difficult to predict and govern. The LLM API is itself important to learn. Attacks are semantic in nature. 

### Security Models Need to Evolve

Traditional API security approaches are not always sufficient for AI systems. Identity propagation, agent permissions, prompt handling, data governance, and outbound access controls all become increasingly important.

### Observability Becomes Critical

AI systems introduce more non-deterministic behavior into distributed systems. Strong observability practices become essential for debugging latency, failed agent workflows, and inference bottlenecks.

### Kubernetes Remains the Operational Foundation

Despite all the excitement around AI frameworks and models, Kubernetes continues to be the operational backbone enabling scalable deployment, networking, and governance for enterprise AI systems.

---

## Community Feedback 

From my discussions and the survey results, people REALLY want hands-on training and are willing to give up a weekend to find it. I intend to do more here. What's my motive? I'm *forced* to learn this stuff in order to teach it.

---

## Looking Ahead

This training day reinforced how quickly AI infrastructure engineering is evolving. It's incredibly overwhelming, even for those of us who have been in the industry for decades.

I think I will focus a future event at a more advanced persona - the K8s-fluent AI engineer who already has the 200-level knowledge, perhaps the 300-level knowledge. I recognize that beginners to the industry need to learn the foundations also, but I am not sure I can keep teaching K8s to a new crop of beginners every session. I'll try to find a balance.

Thanks again to everyone who attended and helped make the event successful.

See you at the next one.

<figure>
    <a href="/assets/k8s-training-day-2/class-selfie.jpg"><img src="/assets/k8s-training-day-2/class-selfie.jpg"></a>
    <figcaption>Class Selfie</figcaption>
</figure>

---