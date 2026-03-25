---
layout: single
title:  "Using ExternalDNS with F5 CIS to Automate DNS on Non-F5 DNS Servers"
categories: [kubernetes]
tags: [kubernetes, f5]
excerpt: "Short overview of one way to use ExternalDNS project with F5 CIS" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/cis-externaldns/cis-externaldns-header.png"><img src="/assets/cis-externaldns/cis-externaldns-header.png"></a>
    <figcaption></figcaption>
</figure>
 
## Overview
 
F5 Container Ingress Services (CIS) is a powerful way to manage BIG-IP configuration directly from Kubernetes. Using CIS Custom Resource Definitions (CRDs) like `VirtualServer` and `TransportServer`, you can define rich traffic management policies in native Kubernetes manifests and have CIS automatically create and update Virtual IPs (VIPs) on BIG-IP.
 
One common question that comes up: **"What if I want DNS records created automatically when a VirtualServer comes up, but I'm not using F5 DNS?"**
 
This article answers exactly that question. We'll walk through how to combine CIS `VirtualServer` resources with the community project [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) to automatically register DNS records on external DNS providers like AWS Route 53, Infoblox, CoreDNS, Azure DNS, and others — all without touching a zone file by hand.

---
 
## Background: How DNS Automation Typically Works in Kubernetes
 
Before diving into the solution, it's worth grounding ourselves in how DNS automation normally works in Kubernetes.
 
### The Standard Pattern: Services of Type LoadBalancer
 
The most common pattern is:
 
1. You create a `Service` of type `LoadBalancer`.
2. A cloud controller (or a bare-metal equivalent like MetalLB) assigns an external IP and updates the `status.loadBalancer.ingress` field of the Service object.
3. ExternalDNS watches for Services of type `LoadBalancer` with specific annotations, reads the IP from the `status` field, and creates a DNS A record on your external DNS server.
 
This is clean, well-understood, and widely supported. ExternalDNS can also watch `Ingress` objects or `Services` of type `ClusterIP` and `NodePort`, but the `LoadBalancer` pattern is by far the most common integration point.
 
### Where F5 CIS Fits In
 
CIS supports creating VIPs on BIG-IP in multiple ways:

1. **VirtualServer / TransportServer CRDs** — Most customers prefer to use [VS or TS CRDs](https://clouddocs.f5.com/containers/latest/userguide/crd/) because they expose richer BIG-IP capabilities: iRules, custom persistence profiles, health monitors, TLS termination policies, and more. This is where the DNS automation story gets more nuanced and is **the focus of this article**.

2. **Service of type LoadBalancer** — CIS watches for Services of type `LoadBalancer`. Typically an IPAM controller or a [custom annotation](https://clouddocs.f5.com/containers/latest/userguide/loadbalancer/#service-type-loadbalancer-annotations) will be used to configure an IP address. CIS allocates a VIP on BIG-IP, and updates the Service's `status.loadBalancer.ingress` field with that IP. This is **not** the focus of this article.

3. **Other** — CIS can also use `Ingress` or `ConfigMap` resources, but these are out of scope for this article.

### The Gap: F5 CRDs and Non-F5 DNS
 
CIS does include its own [ExternalDNS CRD](https://clouddocs.f5.com/containers/latest/userguide/crd/externaldns.html) (not to be confused with the community project of the same name). However, **F5's built-in ExternalDNS CRD only supports F5 DNS (BIG-IP DNS / GTM)**. If you're using Route 53, Infoblox, PowerDNS, or any other DNS provider, you need a different approach.
 
That's where the community ExternalDNS project comes in.
 
---
 
## The Solution: VirtualServer + Service of Type LoadBalancer + ExternalDNS
 
The trick is straightforward once you see it:
 
> CIS can manage a VIP on BIG-IP via a `VirtualServer` CRD while simultaneously updating the `status` field of a `Service` of type `LoadBalancer`. ExternalDNS then reads that `status` field and creates DNS records.
 
Here's the flow:
 
```
VirtualServer CRD
      │
      │  references pool members via
      ▼
Service (type: LoadBalancer)
      │
      │  CIS updates status.loadBalancer.ingress
      │  with the VIP IP address
      ▼
ExternalDNS watches Service status
      │
      │  creates/updates DNS record
      ▼
External DNS Server (Route 53, Infoblox, etc.)
```
 
Let's walk through the manifests.
 
---
 
## Step-by-Step Walkthrough
 
### Step 1: Deploy Your Application
 
A standard Deployment — nothing special here.
 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          ports:
            - containerPort: 8080
```
 
### Step 2: Create the Service of Type LoadBalancer
 
This Service is the linchpin of the whole solution. It serves three purposes:
 
1. It acts as a target for the CIS `VirtualServer` pool (either via NodePort or directly to pod IPs in cluster mode).
2. CIS updates its `status.loadBalancer.ingress` field with the BIG-IP VIP address.
3. ExternalDNS reads its `status` and annotations to create a DNS record.
 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  namespace: my-namespace
  annotations:
    # ExternalDNS annotation — tells ExternalDNS what hostname to register
    external-dns.alpha.kubernetes.io/hostname: myapp.example.com
    # Optional: set a custom TTL
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: LoadBalancer
  # Prevent other LB controllers from acting on this Service
  loadBalancerClass: f5.com/bigip
  # Do not allocate NodePort endpoints — more on this below
  allocateLoadBalancerNodePorts: false
```
 
Two fields here deserve extra explanation.
 
#### `loadBalancerClass: f5.com/bigip`
 
In a typical cluster, multiple controllers may be watching for Services of type `LoadBalancer` — MetalLB, the cloud provider controller, etc. If you're using CIS `VirtualServer` CRDs to manage the VIP (rather than having CIS act directly as a LoadBalancer controller for this Service), you likely don't want any of those other controllers touching this Service.
 
Setting `loadBalancerClass` to a value that no other running controller claims means this Service will be **ignored by all LB controllers except the one that explicitly handles that class**. In this pattern, no controller is assigning the VIP from the Service side — CIS does it via the `VirtualServer` CRD and writes back the IP into the `status` field programmatically.
 
> **Note:** The exact value of `loadBalancerClass` depends on your environment. The key goal is to prevent unintended controllers from assigning IPs or creating cloud load balancers for this Service.
 
#### `allocateLoadBalancerNodePorts: false`
 
By default, `LoadBalancer` Services in Kubernetes allocate NodePort endpoints. This means traffic _could_ reach your pods directly via `<NodeIP>:<NodePort>` — bypassing BIG-IP entirely, bypassing your security policies, and bypassing your iRules.
 
Setting `allocateLoadBalancerNodePorts: false` prevents this. The Service effectively behaves like a `ClusterIP` service in terms of access — the only way to reach it from outside the cluster is via the BIG-IP VIP. This is the right posture when:
 
- Your CIS deployment uses `--pool-member-type=cluster`, sending traffic directly to pod IPs via the BIG-IP's overlay network (VXLAN or GENEVE).
- You want BIG-IP to be the sole external entry point for policy enforcement.
 
### Step 3: Create the VirtualServer CRD
 
Now we define the `VirtualServer`. Note how it references the Service by name in the pool configuration:
 
```yaml
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: my-app-vs
  namespace: my-namespace
  labels:
    f5cr: "true"
spec:
  host: myapp.example.com
  ipamLabel: prod          # Optional: use F5 IPAM Controller for IP allocation
  # virtualServerAddress: "10.1.10.50"  # Or specify IP directly
  pools:
    - path: /
      service: my-app-svc
      servicePort: 80
      monitor:
        type: http
        send: "GET / HTTP/1.1\r\nHost: myapp.example.com\r\n\r\n"
        recv: ""
        interval: 10
        timeout: 10
```
 
When CIS processes this `VirtualServer`, it:
 
1. Creates a VIP on BIG-IP (using either the IP you specified in `virtualServerAddress` or one allocated by the F5 IPAM Controller if `ipamLabel` is used).
2. Configures the BIG-IP pool with the backends from `my-app-svc`.
3. **Writes the VIP IP address back into `my-app-svc`'s `status.loadBalancer.ingress` field.**

{:.notice--info}
That last step is what makes the whole chain work.
 
#### IP Address: Specify Directly or Use F5 IPAM Controller
 
You have two options for IP allocation:
 
**Option A — Specify the IP directly in the VirtualServer manifest:**
 
```yaml
spec:
  virtualServerAddress: "10.1.10.50"
```
 
This is simple and predictable. Good for static, well-planned deployments.
 
**Option B — Use the F5 IPAM Controller:**
 
```yaml
spec:
  ipamLabel: prod
```
 
The [F5 IPAM Controller](https://github.com/F5Networks/f5-ipam-controller) watches for CIS resources with `ipamLabel` annotations and allocates IPs from a configured range. CIS then picks up the allocated IP automatically. This is ideal when you want full automation without managing IP addresses in YAML files.
 
### Step 4: Verify CIS Updates the Service Status
 
After CIS processes the `VirtualServer`, check the Service:
 
```bash
kubectl get svc my-app-svc -n my-namespace -o jsonpath='{.status.loadBalancer.ingress}'
```
 
You should see output like:
 
```json
[{"ip":"10.1.10.50"}]
```
 
This is the IP that ExternalDNS will use to create the DNS record.
 
### Step 5: ExternalDNS Does Its Job
 
With ExternalDNS deployed and configured for your DNS provider (Route 53, Infoblox, etc.), it will:
 
1. Discover `my-app-svc` because it's of type `LoadBalancer` with an `external-dns.alpha.kubernetes.io/hostname` annotation.
2. Read `10.1.10.50` from `status.loadBalancer.ingress`.
3. Create an A record: `myapp.example.com → 10.1.10.50`.
 
ExternalDNS handles the rest automatically, including updates if the IP changes.
 
A minimal ExternalDNS deployment for Route 53 would look like:
 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.0
          args:
            - --source=service
            - --domain-filter=example.com
            - --provider=aws
            - --aws-zone-type=public
            - --registry=txt
            - --txt-owner-id=my-cluster
```
 
Refer to the [ExternalDNS documentation](https://github.com/kubernetes-sigs/external-dns) for provider-specific configuration (IAM roles for Route 53, credentials for Infoblox, etc.).
 
---
 
## Putting It All Together: Summary of the Architecture
 
<figure>
    <a href="/assets/cis-externaldns/cis-externaldns-diagram.png"><img src="/assets/cis-externaldns/cis-externaldns-diagram.png"></a>
    <figcaption>High-level diagram of how the IP address is populated in the Status of the LoadBalancer service and then used by ExternalDNS</figcaption>
</figure>
 
## Key Considerations and Design Choices
 
### When to Use This Pattern vs. CIS as a LoadBalancer Controller
 
CIS _can_ act directly as a LoadBalancer controller — watching Services of type `LoadBalancer` and creating VIPs on BIG-IP without any `VirtualServer` CRD involvement. If that's sufficient for your needs, it's simpler. ExternalDNS works with that mode natively, since CIS updates `status.loadBalancer.ingress` in both cases.
 
Use the `VirtualServer` CRD approach when you need:
- Custom iRules or iApps on the VIP
- Advanced persistence profiles
- Fine-grained TLS termination control
- Traffic splitting or A/B routing policies
- Any BIG-IP capability that doesn't map directly to Kubernetes Service semantics
 
### `allocateLoadBalancerNodePorts: false` — When It Applies
 
This setting is appropriate when your CIS deployment uses `--pool-member-type=cluster`. In cluster mode, BIG-IP sends traffic directly to pod IPs, not through NodePort endpoints. Disabling NodePort allocation:
 
- Prevents back-door access to your application via `<NodeIP>:<NodePort>`.
- Reduces iptables rule sprawl on your nodes.
- Aligns with a clean security boundary where BIG-IP is the sole ingress.
 
If your CIS deployment uses `--pool-member-type=nodeport`, you should **not** set `allocateLoadBalancerNodePorts: false`, as CIS will need those NodePorts to forward traffic.
 
### F5 IPAM Controller Integration
 
The F5 IPAM Controller pairs particularly well with this pattern. Rather than managing VIP IP addresses in your `VirtualServer` manifests, IPAM handles allocation from a configured pool. This means:
 
- Platform teams manage IP ranges in the IPAM controller config.
- Application teams simply specify an `ipamLabel` in their `VirtualServer` manifest.
- CIS picks up the IPAM-assigned IP and writes it to the Service `status` automatically.
 
The ExternalDNS chain remains identical regardless of whether the IP comes from IPAM or is statically assigned.
 
---
 
## Frequently Asked Questions
 
**Q: Can I use this pattern with TransportServer CRDs instead of VirtualServer?**
 
Yes. CIS similarly updates the `status` of a referenced Service when using `TransportServer`. The same approach applies.
 
**Q: What if I want ExternalDNS to also create a CNAME instead of an A record?**
 
Use the `external-dns.alpha.kubernetes.io/target` annotation on the Service to override the IP with a hostname, causing ExternalDNS to create a CNAME. Refer to ExternalDNS documentation for specifics.
 
**Q: Can I use multiple hostnames for the same VirtualServer?**
 
Add multiple `external-dns.alpha.kubernetes.io/hostname` annotations (comma-separated values are supported by ExternalDNS) or create additional Services pointing to the same pods.
 
---
 
## Conclusion
 
Combining F5 CIS `VirtualServer` CRDs with the community ExternalDNS project gives you the best of both worlds: rich BIG-IP traffic management via CIS, and flexible, provider-agnostic DNS automation via ExternalDNS.
 
The core insight is simple — **CIS writes the BIG-IP VIP IP address back into the Kubernetes Service `status` field, and ExternalDNS reads from that same field**. By using `loadBalancerClass` and `allocateLoadBalancerNodePorts: false`, you ensure the Service is a clean "status carrier" that doesn't accidentally expose your application through unintended paths.
 
Whether you assign VIP IPs statically in your manifests or use the F5 IPAM Controller for full automation, this pattern integrates naturally into any Kubernetes-native GitOps workflow.
 
---
 
## Additional Resources
 
- [F5 CIS Documentation](https://clouddocs.f5.com/containers/latest/)
- [F5 CIS VirtualServer CRD Reference](https://clouddocs.f5.com/containers/latest/userguide/crd/virtualserver.html)
- [F5 IPAM Controller on GitHub](https://github.com/F5Networks/f5-ipam-controller)
- [ExternalDNS on GitHub](https://github.com/kubernetes-sigs/external-dns)
- [ExternalDNS: Service Source Documentation](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/sources/service.md)
- [Kubernetes: LoadBalancer Service specification](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
 