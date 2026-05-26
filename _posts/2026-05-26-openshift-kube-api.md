---
layout: single
title: "Exposing Openshift API server"
date: 2026-05-26
categories: ["openshift"]
tags: ["openshift, kubernetes"]
excerpt: "How to expose kube-apiserver in Openshift" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

![Exposing OpenShift API with F5 BIG-IP](/assets/openshift-api-server/ocp-api-header-image.png)

When integrating F5 CIS with OpenShift, a common requirement is exposing internal `ClusterIP` services externally through BIG-IP virtual servers. For most services, CIS works exactly as expected, however, the Kubernetes API server is special. 

### The Two API Services in OpenShift
OpenShift exposes the Kubernetes API server through two Services:

| Service      | Namespace                  | Purpose                                  |
| ------------ | -------------------------- | ---------------------------------------- |
| `kubernetes` | `default`                  | Standard upstream Kubernetes API access  |
| `apiserver`  | `openshift-kube-apiserver` | OpenShift infrastructure/operator access |

**Both ultimately reach the same kube-apiserver instances running on control plane nodes.**
{: .notice--success}

The implementation details, however, are very different.

#### The default/kubernetes Service Is Special
The upstream `kubernetes.default.svc` does not use selectors.

You can verify this:
```bash
oc get svc kubernetes -n default -o yaml
```
Notice there is no selector:
```bash
spec:
  clusterIP: 172.30.0.1
  ports:
  - port: 443
    targetPort: 6443
```
This Service exists to solve a bootstrap problem:

How can workloads reliably discover the API server before normal Kubernetes controllers are fully operational?
{: .notice--success}

As explained in [this excellent article from NetworkOP](https://networkop.co.uk/post/2020-06-kubernetes-default/)]:

*"…the endpoints for default/kubernetes are populated through special internal reconciliation logic rather than standard selector-based endpoint management."*

#### Why This Matters for F5 CIS
F5 CIS expects Kubernetes Services to behave conventionally:
```bash
Service
  -> Selector
     -> Pods
        -> EndpointSlices
           -> BIG-IP Pool Members
```
This works perfectly for most application workloads. However, default/kubernetes breaks several assumptions:

- no selector
- manually managed endpoints
- special reconciliation behavior
- not intended for generic ingress exposure

The issue is not that the Service lacks endpoints. The issue is that the Service was never designed to function as a standard externally exposed Service.

### OpenShift Provides a Better Alternative
OpenShift introduces: `openshift-kube-apiserver/apiserver`
<br><br>
Unlike default/kubernetes, this Service behaves much more like a standard Kubernetes Service. Inspect it:

```bash
oc get svc apiserver -n openshift-kube-apiserver -o yaml
```
You will typically observe:

- selector-backed behavior
- standard EndpointSlice reconciliation
- integration with OpenShift service CA management

This makes it far more compatible with CIS.

#### Important Clarification

The kube-apiserver instances are still static control-plane pods managed by the cluster-kube-apiserver-operator. 

This is not a normal Deployment. The distinction is not about the backend pods themselves. The distinction is about:

- how the Services are reconciled
- how endpoints are managed
- which consumers are expected to use them

### We Do Not Want Internal API Traffic Through BIG-IP
BIG-IP should not sit in the middle of Kubernetes east-west control-plane traffic. The goal is only to expose the API externally.

Recommended model:

| Traffic Type              | Path                      |
| ------------------------- | ------------------------- |
| Internal cluster traffic  | Native cluster networking |
| kubelet -> apiserver      | Native networking         |
| operator -> apiserver     | Native networking         |
| External admin/API access | BIG-IP                    |
| External console access   | BIG-IP                    |

BIG-IP is excellent for:

- external HA
- TLS termination
- WAF
- external load balancing

…but internal control-plane communication should remain native.

### Recommended Architecture 
```bash
External Users
        |
        v
+-------------------+
|     F5 BIG-IP     |
|  Virtual Server   |
+-------------------+
        |
        v
+-------------------------------+
| openshift-kube-apiserver      |
| Service: apiserver            |
+-------------------------------+
        |
        v
+-------------------------------+
| kube-apiserver static pods    |
| on control plane nodes        |
+-------------------------------+

Internal cluster traffic bypasses BIG-IP
```

### Final Thoughts
The default/kubernetes Service is one of Kubernetes’ oldest and most specialized abstractions. It exists primarily for bootstrap reliability and universal API discovery.

It was not designed as a generic ingress target.

When exposing the Kubernetes API externally through F5 CIS, the better option in OpenShift is usually:

```bash
openshift-kube-apiserver/apiserver
```

This allows BIG-IP and CIS to interact with a more conventional Service abstraction while keeping internal control-plane communication native to the cluster network.