---
layout: single
title:  "Consolidating multiple Azure services with BIG-IP"
categories: [azure]
tags: [azure,f5]
excerpt: "In this real-world case, these Azure services were less performant, more limited, more expensive than an enterprise solution like BIG-IP." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

### Summary
This customer architecture pieced together Azure Firewall, Azure App Gateway (AAG), and Azure API Management (APIM). Consolidating all of these functions on F5 BIG-IP saw more features, faster throughput, and less cost.

### Architecture with Azure services
This customer came with a well-defined problem statement. Their public-facing API required some WAF and API protections, and they had built this using 3x Azure native services below:

<figure>
    <a href="/assets/azure-fw-apim-aag/customer_architecture.png"><img src="/assets/azure-fw-apim-aag/customer_architecture.png"></a>
    <figcaption>This is the actual diagram the customer had for their environment.</figcaption>
</figure>

#### Problems with this architecture
This architecture works, but suffers multiple problems at enterprise scale. Problems:
1. **Performance**. The customer reported that after the Azure App GW with WAF had more than 40 paths, throughout dramatically slowed.
2. **Scale**. Given the above performance observations, the customer was not confident scaling to enterprise levels. Each Azure service (firewall, AAG, APIM) required further configuration to scale (usually adding to the cost).
3. **Cost**. This one is straight from the customer: "The Azure Firewall is very very expensive". 

Those problems are easily measurable, but there were other issues that also affected their project's success:
- *Complexity* - the complexity and sprawl of the Azure services left most team members unclear on overall architecture and responsibilities
- *Operations* - the disparate Azure services required non-trivial expertise from several teams, yet no single team had overarching cloud architecture knowledge.
- *Enterprise-level feature set* - some additional features that the customer was expecting were not delivered by Azure services. For example, the ability to perform packet tracing, robust logging and telemetry to multiple supported third parties, advanced TLS configurations, etc.

#### How did we even end up here?
This is a common-enough story. 

The Azure Function App was built to serve API calls, but had to be exposed via some kind of WAF protection. Azure App Gateway was used first, in order to apply WAF to inbound API calls. After about 40 paths, throughput became very limited.

In an attempt to work around this, APIM was configured. Multiple traffic paths from AAG were consolidated by batching traffic and sending to APIM. However, APIM is very expensive.

There is organizational policies that require all inbound traffic to traverse Azure Firewall, as well as outbound traffic.




