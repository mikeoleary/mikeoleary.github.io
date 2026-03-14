---
layout: single
title:  "Remote CR discovery with multi-cluster CIS"
categories: [kubernetes]
tags: [kubernetes, f5]
excerpt: "Short overview of new feature in multi-cluster CIS" #this is a custom variable meant for a short description to be displayed on home page
---
<figure>
    <a href="/assets/cis-multicluster/cis-multicluster-header.png"><img src="/assets/cis-multicluster/cis-multicluster-header.png"></a>
    <figcaption></figcaption>
</figure>

### Remote CR discovery for CIS multi-cluster

#### Background
I've watched one of my favorite projects at F5, CIS (Container Ingress Services), mature over the years. When I joined F5 in late 2018, it watched the K8s API for the creation of Ingress resources and configured a VirtualServer on the BIG-IP with settings determined by the annotations on the Ingress resource. Then I watched over the years:
  - it could send AS3 declarations to BIG-IP based on ConfigMap resources
  - it supported Custom Resources (CRD's) like `VirtualServer` and `TransportServer`
  - the CRD schema grew and matured to support almost all available BIG-IP settings
  - it integrated with IPAM and DNS
  - it started supporting pool members using pods from multiple clusters

#### Remote CR discovery
What's the latest improvement? Until now, even when operating in multi-cluster mode, CIS only discovered Custom Resources that were installed *on the same cluster on which CIS was running*. While pods from remote clusters could become pool members in BIG-IP, the `VirtualServer` and `TransportServer` resources had to be on the same cluster as CIS.

Why is this a problem? 

Imagine you have 4 clusters and they all run different parts of the same overall application. You may have multiple clusters for the sake of: 
  - **redundancy** (in case of cluster failure)
  - **migration** (you're moving workloads between namespaces and clusters)
  - **cost reasons** (some workloads on more costly clusters)
  - etc

You may choose to install CIS on Cluster 1 and have CIS "watch" all 4 clusters for services to expose via BIG-IP. But before now, you had to install CR's - `VirtualServer` and `TransportServer` - on Cluster 1 only. That means you have to control where your app teams deploy their resources. 

Now, CIS will watch all 4 clusters for services AND Custom Resources. That's the feature introduced here.

#### Before remote CR discovery

<figure>
    <a href="/assets/cis-multicluster/cis-multicluster-diagram1.png"><img src="/assets/cis-multicluster/cis-multicluster-diagram1.png"></a>
    <figcaption>With multi-cluster, without remote CR discover</figcaption>
</figure>

#### With remote CR discovery

<figure>
    <a href="/assets/cis-multicluster/cis-multicluster-diagram2.png"><img src="/assets/cis-multicluster/cis-multicluster-diagram2.png"></a>
    <figcaption>With multi-cluster, with remote CR discovery</figcaption>
</figure>

### How to configure remote CR discovery
As you can see from the diagrams, the difference is quite simple, but the operational impacts are large. Enabling this is equally simple: a single line in the extended spec config file (see line 15 below)

````yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: extended-spec-config
  namespace: kube-system
  labels:
    f5type: virtual-server
    as3: "true"
data:
  extendedSpec: |
    mode: default
    externalClustersConfig:
    - clusterName: cluster2
      secret: kube-system/kubeconfig2
      customResourceDiscovery: true #enables this new feature
````

For the sake of full repeatability, I am going to share my manifests [here](https://github.com/mikeoleary/cis-multicluster-remote-cr-discovery).

Thanks for reading!