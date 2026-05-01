---
layout: single
title: "Foundational engineers and the Abstraction Debt"
date: 2026-04-29
categories: [ai]
tags: [ai]
excerpt: "Some thoughts on the pros and cons of learning fundamentals vs moving fast" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/foundational-engineers/foundational-engineers-header.png"><img src="/assets/foundational-engineers/foundational-engineers-header.png"></a>
    <figcaption>Foundational complexity vs abstractions - where do you land?</figcaption>
</figure>
Recently I've been feeling an instinctive alarm bell sound as I feel fundamental technology skills slipping away in favor of moving faster with AI. At this point I'm not judging bad vs good, but in technology, AI helps us move faster while it frees us from learning *how* we did this.

> There is a pattern in systems engineering that repeats itself across every generation of tooling: we build an abstraction that is brilliant at hiding complexity, celebrate the productivity gains, and then spend the next decade paying back the debt that abstraction quietly accumulated.

I recently felt this again when reading an [example](https://www.tomshardware.com/tech-industry/artificial-intelligence/claude-powered-ai-coding-agent-deletes-entire-company-database-in-9-seconds-backups-zapped-after-cursor-tool-powered-by-anthropics-claude-goes-rogue) agent tooling with catastrophic consequences. Anthropic's MCP specification deserves the "USB-C for AI" metaphor. It defines a clean, universal interface between an AI host, a language model, and the external world — local filesystems, databases, APIs, Kubernetes clusters. If you can expose a tool over JSON-RPC, the LLM can use it.

My instinctive alarm bells are set off by a new factor: non-deterministic systems making decisions. I used to feel that abstractions just allowed for faster development at the cost of foundational learning. Now I feel we have a combination of:
- abstractions making development faster (fewer foundational skills)
- AI making coding accessible to beginners (vibecoding)
- frameworks like MCP mass-enabling interaction with external systems (real world consequences)
- AI models being non-deterministic (intelligent guessing)
- Natural language prompts (requiring semantic reasoning from LLMs)

Each factor builds on the others, but it's the combination of intelligent guessing and real-world consequences that summarizes my concern. The vibecoding and "speed over accuracy" culture adds to the concern.

---

## The Traffic Flow Nobody Is Auditing

Before I get philosophical, it's worth being precise about what actually happens when an MCP-enabled agent runs a request. The flow is:

```
Host Application
    │
    ├─► LLM (probabilistic reasoning layer)
    │       │
    │       └─► Tool Call Selection  ← "intelligent guess"
    │
    └─► MCP Server (executes side effects)
            │
            ├─► Filesystem operations
            ├─► API calls
            ├─► Database writes
            └─► Shell commands
```

Notice the component in the middle: *probabilistic reasoning*. The LLM does not parse a deterministic rule to decide which tool to invoke. It reasons about the *semantics* of the available tools — their names, their descriptions, their parameter schemas — and makes an inference about which one best serves the user's intent. It is, in the most literal sense, an educated guess.

In isolation, that's fine. In combination with tools that carry real-world side effects, it's concerning. Especially when the tools themselves may be vibecoded (potentially low quality) and the natural language input will be imperfect.

---

## Semantic Access Is Not Deterministic Control

Here's an example I fear will happen with home-grown MCP servers for BIG-IP, for example.

When you configure an F5 BIG-IP, you work with objects that have precise, non-overlapping definitions. A *VIP* (Virtual IP) is the front-end address that external clients connect to. A *virtual server* is the F5 configuration object that listens on that VIP and applies policies. These terms are close enough in everyday language that a reasonable person — or a language model — might use them interchangeably. In the F5 management plane, they are not interchangeable. Conflating them means you are operating on the wrong object, applying the wrong policy, and potentially taking down traffic for a production service while believing you are doing something benign.

Now expose your network automation layer as MCP tools. Name one `get_virtual_server_config` and another `get_vip_status`. Ask an LLM agent to "check the load balancer for the payments service." Which tool does it call? It depends entirely on which description the model finds semantically closer to "load balancer" and "check." There is no compilation step to catch the mismatch. There is no type system enforcing the distinction. There is a probability, and a reliance on the description of the tools learned by the model.

This is what I mean by *Abstraction Debt* — when human operators lose understanding of the systems they govern. The debt is not visible in normal operations. It becomes visible when you need it: during an incident, when you are trying to understand what the agent actually did and why, when you are trying to teach others, etc.

---

## Two Classes of Engineer Are Emerging

<figure>
    <a href="/assets/foundational-engineers/foundationalists.png"><img src="/assets/foundational-engineers/foundationalists.png"></a>
    <figcaption>This is just my current thinking. Perhaps it's not a spectrum but a buffet of approaches we choose from daily. I think my goal is to find the right balance, not necessarily to choose a camp.</figcaption>
</figure>

Again, this is Claude helping me build my thoughts into a case:

**Orchestrators** assemble systems using semantic reasoning. They write prompts, configure MCP servers, chain tool calls, and build agents that can traverse a multi-hop workflow across half a dozen services. They are productive at a pace that would have seemed impossible five years ago. They are, right now, extremely employable.

**Foundationalists** understand fundamentals. They know what happens at the TCP level when a JSON-RPC request leaves the MCP client. They can read a packet capture during an incident. They understand process management well enough to know why a forked subprocess inherited the wrong file descriptors. They can reason about the OSI model when an abstraction leaks and the stack trace is unhelpful.

The risk is not that Orchestrators exist. The risk is that we're incentivizing engineers to become *only* Orchestrators — and that the feedback loops which historically taught foundational knowledge (debugging, reading logs, understanding failure modes) are being bypassed by systems that hide their internals behind a semantic interface.

The demand curve for Orchestrators will remain high until the first major incident at scale: a misconfigured MCP server with filesystem access, a semantically confused tool call that mutates production data, an agent that "helpfully" applies a Kubernetes manifest to the wrong namespace because the cluster names were similar enough in the model's embedding space. At that moment, the Orchestrator reaches for their incident playbook and finds it empty. The Foundationalist is already reading the audit logs.

---

## Knowledge Atrophy

(Again, somewhat ironically, Claude has helped me refine the following thoughts.)

A claim made by others but I believe in: *when automation handles a task reliably, the human practitioner stops practicing the underlying skill*. The skill atrophies. When the automation fails, the human is less capable of recovering than they would have been without the automation.

MCP is a powerful automation layer systems integration. When it works, it removes the need to understand transport protocols, to handle authentication edge cases, to reason about retry semantics. Engineers who use it exclusively will, over time, become less capable of debugging systems at the level where those concerns live. The protocol doesn't teach you what it hides.

This is not an argument against abstraction. Abstraction is the mechanism by which the industry makes progress. It is an argument for *deliberate practice at the layer beneath the abstraction you rely on*. The engineers who built the most reliable systems I have worked with could always go one layer deeper than the tooling required. They understood TCP because they had debugged socket timeouts. They understood HTTP because they had read raw request logs. That reservoir of foundational knowledge is what gets drawn on when the abstraction leaks.

MCP makes it easy to never build that reservoir. That is the danger.

---

## Where the Abstraction Leaks

MCP's abstraction is not watertight. Here are the specific seams where it fails, ranked roughly by incident frequency:

**1. Tool Description Drift**  
Tool descriptions are the primary signal the LLM uses for selection. As tools are updated, renamed, or repurposed, their descriptions often lag. The model continues reasoning against stale semantic signals. There is no schema enforcement to catch this.

**2. Implicit State Assumptions**  
MCP tools are stateless by design at the protocol level. But the systems they connect to are stateful. A tool that "lists files in a directory" and a tool that "moves files to archive" share no transactional context within the MCP call graph. An agent that calls both in sequence across a failure boundary may leave the system in an inconsistent state that neither the agent nor the operator anticipated.

**3. Authentication Context Collapse**  
In a traditional system, the principal making a request is explicit: a service account, a user token, an IAM role. In an MCP-enabled agent flow, the effective principal is often the host process. The LLM's reasoning — which is untraceable in any cryptographic sense — is the de facto access control layer. This is not a security model. It is the absence of one.

**4. Error Signal Ambiguity**  
When an MCP tool call fails, the LLM receives a text error message and reasons about what to do next. It may retry with different parameters. It may call a different tool. It may hallucinate a recovery strategy. None of this is logged in a way that a standard SIEM or observability platform can parse without custom instrumentation.

---

## The Call to Action

The engineers who will be valuable in five years are not the ones who orchestrated the most agent workflows. They are the ones who understand why those workflows fail, can trace a failure through the full stack, and can reason about the system's behavior at the level of the protocol rather than the semantic description.

The path is not to avoid MCP. It is to use it with deliberate awareness of what it hides, and to maintain active practice in the disciplines it makes it easy to skip.

Concretely:

- Read the MCP specification transport layer documentation. Understand that stdio and SSE/HTTP are meaningfully different transport modes with different security boundaries.
- When something goes wrong in an agent workflow, resist the urge to re-prompt. Capture the raw tool call log and trace it manually before asking the model to self-correct.
- Build at least one MCP server from a bare HTTP server before using a framework. The framework abstracts the right things, but you need to know what is being abstracted.
- Keep your Wireshark skills current. JSON-RPC over HTTP is readable in a packet capture. Read it.
- Study your infrastructure tools at the API level, not just through the semantic lens an agent provides. The F5 API, the Kubernetes API, the AWS SDK — the objects in these systems have precise definitions that semantic reasoning does not respect.

The Foundationalist who orchestrates is not just more resilient than the pure Orchestrator. They are the person in the room when the incident happens and the AI-built system has failed in a way no one anticipated. That person will not be replaced by an AI agent. They will be the one telling the AI agent what actually went wrong.

Stay foundational.

---