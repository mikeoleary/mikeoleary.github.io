---
layout: single
title:  "Introducing PayGate"
categories: ["big-ip"]
tags: ["big-ip"]
excerpt: "Monetize premium API access at the edge with zero backend changes." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/paygate/paygate-header-image.png"><img src="/assets/paygate/paygate-header-image.png"></a>
</figure>

*Developers don’t always want to modify their backend apps just to charge for premium API features.*


### Summary
PayGate is a powerful iRule for F5 BIG-IP that **implements pay-to-access functionality entirely at the network edge**, with *zero backend changes required*.


---

### Why PayGate?

- **Instant monetization** of API endpoints — no code changes, no redeploys  
- **Flexible entitlement** via tokens stored in BIG-IP’s memory  
- **Edge-enforced security**: protected webhook, token validation, TTL handling  
- **Self-contained operation**: no external dependencies like Redis or iRulesLX  
- **Business-aligned**: measure demand, treat price as signal, and unlock revenue in *hours*


### Architectural overview
<figure>
    <a href="/assets/paygate/paywall-iRule-diagram.png"><img src="/assets/paygate/paywall-iRule-diagram.png"></a>
</figure>

---

### 🧩 How It Works

#### 1. Protected Token Injection  
Stripe sends a `checkout.session.completed` webhook → your Python handler which extracts the `entitlement_token` and sends it securely to BIG-IP (`/cache-token`) with a shared secret.

```tcl
# iRule snippet
when HTTP_REQUEST {
  if { [HTTP::path] eq "/cache-token" && [HTTP::header "X-Webhook-Secret"] eq $static::shared_token_secret } {
    # save token into table
  }
}
```

#### 2. Table-Based Caching
Tokens are stored in a BIG-IP `table` with a 1-hour TTL. No Redis or external caching — it's all in-memory on BIG-IP.

#### 3. Edge-Gated API Requests
When the API client requests `/api/v1/hyperlocal?entitlement_token=...`, the iRule:
- Extracts the token
- Looks it up in the table
- If found → injects a header (X-Entitlement-Tier) and allows the request
- If missing or expired → redirects to /pricing, where users can pay again

### 🛠️ Demo & Code
- GitHub Repo: [mikeoleary/paywall-irule-demo](https://github.com/mikeoleary/paywall-irule-demo)
- Complete iRule with webhook gateway, token cache, and premium routing
- Python Flask webhook handler to verify Stripe events
- `/token-status` and `/purge-token` endpoints on BIG-IP for monitoring and token cleanup (can be added easily but are not part of this demo)

### 💡 Detailed traffic flow
<figure>
    <a href="/assets/paygate/paywall-iRule.png"><img src="/assets/paygate/paywall-iRule.png"></a>
</figure>

### 💡 Business Benefits

| Feature | Benefit |
| ------------- | ------------- |
| Edge enforcement | No backend modifications needed |
| Real-time monetization | Start charging premium instantly |
| Demand measurement | Paywall conversion = clear product-market fit |
| Rapid iteration | Change pricing, features, or TTLs without app release |
| Cost-efficient | No extra infrastructure, all on existing BIG-IP |

### ✅ What’s Next
- 📈 A/B pricing tests — change price or TTL to optimize conversion
- 📊 Usage analytics — track token issuance and redemption trends
- 📦 Expand features — tier-based access, usage quotas, premium burst credits
- 🔒 Packaging — package as a blueprint or DevCentral article

### 🚀 Get Started
- Clone the repo and load PayGate iRule onto your BIG-IP
- Deploy the Flask webhook on your VM
- Create a Stripe product with entitlement_token metadata
- Route /cache-token, /token-status, and /purge-token to your BIG-IP
- Test the flow: pricing → checkout → API access

With PayGate, you’ve turned your BIG-IP into a revenue-generating edge gateway, empowering API monetization in hours, not months. Let me know what you think!