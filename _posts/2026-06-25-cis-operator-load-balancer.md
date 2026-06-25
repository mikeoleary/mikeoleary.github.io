---
title: "CIS: OpenShift Operator + Service type LB"
layout: single
excerpt: A customer has CIS installed via the operator and a Service of type LoadBalancer. Here's the operator-level config, and the Service annotation example.
categories:
  - openshift
tags:
  - openshift
  - f5
  - cis
  - kubernetes
toc: true
toc_sticky: true
---

<figure>
    <a href="/assets/cis-operator-lb/operator-header2.jpg"><img src="/assets/cis-operator-lb/operator-header-2.jpg"></a>
    <figcaption>Don't forget to drink 8 glasses of water today.</figcaption>
</figure>

In [my last post]({{ site.url }}/openshift/cis-operator-cli/) I installed the F5 CIS operator from the CLI and brought up the `F5BigIpCtlr` operand. This post picks up right after that: the operator is running, CIS is healthy, and now a customer wants to expose an application with a plain `Service` of `type: LoadBalancer` instead of an `Ingress`, a `Route`, or a `VirtualServer` custom resource.

### Why Service Type LB? 
Some teams don't want to manage a CIS-specific CRD for every app - they just want the same `type: LoadBalancer` experience they'd get from a cloud provider's native load balancer, and they want BIG-IP to be the thing that satisfies it on-prem.

The short version: CIS doesn't watch `Service` objects for `LoadBalancer` behavior out of the box. Two operator-level `args` need to be set before CIS will even look at the Service, and then the Service itself needs an annotation telling CIS what IP to use.

### Why a Service alone isn't enough

If you just create a `Service` with `type: LoadBalancer` against a default CIS install, nothing happens. `EXTERNAL-IP` sits at `<pending>` forever, because there's no cloud controller manager assigning it and CIS, by default, isn't watching for this resource type at all.

CIS has to be put into a mode where it watches Custom Resources, and on OpenShift, it needs to know which CNI plugin is in play so it can correctly read pod and node networking information. Both of these are operator-level settings, not something you can fix from the Service manifest alone.

### Notable parameters
Notice these parameters were all included when we defined our operand in our last post.

| Parameter | What it does |
| --- | --- |
| `custom_resource_mode` | Switches CIS into CRD mode. This is also the mode that gives CIS native support for `Service` objects of `type: LoadBalancer` - without it, CIS only processes Routes or ConfigMaps, never Services. |
| `orchestration_cni`    | Tells CIS which CNI plugin the cluster runs, so it can correctly read pod CIDR and node IP information for that CNI. For an OpenShift cluster running OVN-Kubernetes (the default since 4.x), this is `ovn-k8s`. |
| `static_routing_mode`  | Adds Static Routes on the BIGIP so that traffic can be directly route to the pods. (Without tunnels) |

Worth calling out: `custom_resource_mode` is the parameter that's actually mandatory for `Service` type `LoadBalancer` support. `static_routing_mode` and `orchestration_cni` enable routes to be created on F5 BIG-IP so that the BIG-IP can know which Pod IP address blocks belong to which OpenShift nodes.

### The F5BigIpCtlr operand

This is the same operand I had in the last blog post. 

```yaml
apiVersion: cis.f5.com/v1
kind: F5BigIpCtlr
metadata:
  name: cis1
  namespace: kube-system
spec:
  namespace: kube-system
  args:
    bigip_url: 10.0.4.11
    bigip_partition: openshift
    log_level: DEBUG
    insecure: true
    pool_member_type: cluster
    manage_routes: false
    custom_resource_mode: true
    static_routing_mode: true
    orchestration_cni: ovn-k8s
  bigip_login_secret: bigip-login
  image:
    repo: f5networks/cntr-ingress-svcs
    user: registry.connect.redhat.com
    pullPolicy: Always
  version: 2.20.4-ubi10
  ingressClass:
    create: false
    defaultController: false
    ingressClassName: f5
  rbac:
    create: true
    namespaced: false
  serviceAccount:
    create: true

```

### Exposing the application

With CIS configured, the customer's application Service just needs `type: LoadBalancer` plus one CIS-specific annotation that tells CIS which IP to configure on BIG-IP for the virtual server.

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    cis.f5.com/ip: 10.8.3.1 #the VIP gets created with this IP
  labels:
    app: hello-world
  name: hello-world-svc-1
  namespace: hello-world
spec:
  ports:
    - name: port8080
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: hello-world
  type: LoadBalancer
```

Check that the IP landed on the Service:

```bash
oc get svc hello-world-svc-1 -n hello-world 
```

`EXTERNAL-IP` should now show `10.8.3.1` instead of `<pending>`, and CIS will have created a corresponding LTM virtual server on BIG-IP at that address.

### A note on the IPAM alternative
`cis.f5.com/ip` is the hardcoded path - good for a customer who already owns a static IP and just wants CIS to honor it, which is the case I'm documenting here. F5 also ships an [F5 IPAM Controller](https://clouddocs.f5.com/containers/latest/userguide/ipam/) that can assign the address automatically from a pre-defined range, using a `cis.f5.com/ipamLabel` annotation instead of `cis.f5.com/ip`. That path needs the additional `ipam: true` arg on the CIS operand and an `F5IPAM` resource to define the range.

If both annotations happen to be present on the same Service, `cis.f5.com/ip` wins - CIS treats the hardcoded IP as the more specific instruction. For most customer environments I've seen, where IP assignment is already governed by an IPAM process outside the cluster, hardcoding via `cis.f5.com/ip` is the simpler and more predictable choice. IPAM earns its keep once you have many Services and don't want to hand-manage individual addresses.

### Final thoughts

Simple blog post intended for a copy/paste option for a customer that has installed CIS via OpenShift operator and wants to expose services of type `LoadBalancer`.

If you're working through this for a real customer deployment, feel free to reach out!
