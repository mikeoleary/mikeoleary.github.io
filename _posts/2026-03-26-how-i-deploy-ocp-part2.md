---
layout: single
title:  "Deploying OpenShift smarter: GPU nodes for test clusters"
categories: [openshift]
tags: [openshift, f5]
excerpt: "Deploy Openshift on AWS, Installer Provisioned Infrastructure, with GPU node at low cost" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
<figure>
    <a href="/assets/openshift-low-cost/openshift-low-cost-header.jpg"><img src="/assets/openshift-low-cost/openshift-low-cost-header-custom.jpg"></a>
    <figcaption></figcaption>
</figure>
 
This is a follow-up to my [original post]({% post_url 2023-11-01-how-i-deploy-ocp %}) from November 2023 and [another]({%post_url 2024-09-19-deploying-openshift-with-metal-nodes %}) from Sept 2024. A lot has changed since then — not in how OpenShift installs, but in my AWS environment and in some installer requirements that crept in quietly. 

This post covers the problems I hit deploying OCP 4.21 after not doing this since 4.16, and the changes I made to end up with a cheaper, leaner dev/test cluster.

### What broke since last time

It's been 6-12 months since I remember installing Openshift on AWS. Two things had changed that caused immediate failures.

#### 1. Subnet tagging is now required

The installer now validates subnet tags before proceeding. If your subnets are untagged, you'll get a fatal error before it even starts. Previously the installer would just tag your subnets itself — now it checks first.

**Subnets you're handing to the installer**, ie those in your install-config.yaml, need to be tagged:

```bash
aws ec2 create-tags \
  --resources subnet-xxxxxxxxxxxxxxxxx subnet-xxxxxxxxxxxxxxxxx \
  --tags Key=kubernetes.io/cluster/ocpcluster,Value=shared
```

Use your cluster name from `metadata.name` in place of `ocpcluster`.

**Any other subnets in the same VPC** that the installer should ignore need this tag:

```bash
aws ec2 create-tags \
  --resources subnet-xxxxxxxxxxxxxxxxx \
  --tags Key=kubernetes.io/cluster/unmanaged,Value=true
```

If you skip this step, the installer will fail with an error about untagged subnets in the VPC.

#### 2. Corporate IT had tightened my IAM permissions

My AWS credentials hadn't changed, but the policy attached to my IAM user had. The new policy uses a `NotAction` block that explicitly excludes `iam:*User*` and `iam:*AccessKey*` operations (except on my own user). The installer's default **mint mode** for the Cloud Credential Operator (CCO) needs to create and delete IAM users for each cluster component — so it now fails hard with:

```
WARNING Action not allowed with tested creds   action=iam:DeleteAccessKey
WARNING Action not allowed with tested creds   action=iam:DeleteUser
FATAL failed to fetch Cluster: failed to fetch dependency of "Cluster": failed to generate asset "Platform Permissions Check": validate AWS credentials: current credentials insufficient for performing cluster installation
```

The fix is to add `credentialsMode: Passthrough` to your `install-config.yaml`. In passthrough mode, the CCO copies your existing credential to each cluster component instead of minting new scoped IAM users. It never needs `iam:CreateUser` or `iam:DeleteUser`.

```yaml
credentialsMode: Passthrough
```

This is a supported and documented mode, and for a dev/test cluster it's a perfectly reasonable trade-off. The main thing to know is that your AWS credential becomes a long-lived dependency of the cluster — don't rotate or delete it without updating the cluster secret first.

---

### Cost optimizations

The original setup deployed 3 control plane nodes and 3 workers. That's 6 EC2 instances running at all times. For a dev/test cluster this is overkill. Here's what I changed.

#### Drop to 1 master and 1 worker

For dev/test, a single master and single worker is fine. A 3-node control plane gives you etcd quorum tolerance — if you don't care about that (and for a throwaway lab cluster, I don't), 1 master works.

#### Use Spot instances for workers

Workers are safely replaceable. The MachineSet will automatically provision a new one if a Spot interruption occurs. In us-east-1, an `m5.xlarge` Spot instance typically runs 60-70% cheaper than on-demand.

{:.notice--warning}
**Important:** do not use Spot for the master node. If it gets interrupted, the cluster is dead. Keep the master on on-demand.

#### Single availability zone

Running across multiple AZs incurs cross-AZ data transfer costs. For a dev/test cluster, just pick one AZ. This also means you only need 2 subnets (one public, one private) instead of 4.

#### Here's my updated install-config.yaml

```yaml
additionalTrustBundlePolicy: Proxyonly
apiVersion: v1
baseDomain: my-f5.com
credentialsMode: Passthrough
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 1
  platform:
    aws:
      type: m5.xlarge
      spotMarketOptions: {}   # Spot pricing - ~60-70% cheaper than on-demand
      rootVolume:
        size: 100
        type: gp3
      zones:
      - us-east-1a
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 1
  platform:
    aws:
      type: m5.xlarge         # On-demand - spot interruption = dead cluster
      rootVolume:
        size: 100
        type: gp3
      zones:
      - us-east-1a
metadata:
  creationTimestamp: null
  name: ocpcluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: us-east-1
    subnets:
    - subnet-xxxxxxxxxxxxxxxxx   # Private subnet - us-east-1a
    - subnet-xxxxxxxxxxxxxxxxx   # Public subnet  - us-east-1a
publish: External
pullSecret: 'my-pull-secret'
sshKey: |
  ssh-rsa xxx...yyy imported-openssh-key
```

---

### On-demand GPU nodes — deploy when you need them, delete when you don't

I occasionally need to test an app that requires a GPU. Running a GPU instance full-time is expensive — a `g4dn.xlarge` (1x NVIDIA T4) runs about $0.53/hr on-demand, or around $0.17/hr on Spot. The right pattern here is to create a MachineSet for the GPU node after the cluster is up, scale it to 1 when you need it, and scale it back to 0 when you don't.

**When scaled to 0, the EC2 instance is terminated — you pay nothing for compute.** The only ongoing cost is the EBS root volume that remains (~$10/month for 100GB gp3). If that bothers you, delete the MachineSet entirely and recreate it next time.

#### Creating the GPU MachineSet

Use an existing MachineSet as your template:

```bash
MACHINESET_NAME=$(oc get machineset -n openshift-machine-api -o name | head -1)
oc get machineset -n openshift-machine-api -o yaml $MACHINESET_NAME > gpu-machineset.yaml
```

Edit `gpu-machineset.yaml` and make these changes:

- Change the `name` (in both `metadata.name` and the selector labels) to something like `ocpcluster-gpu-us-east-1a`
- Set `replicas: 0`
- Set `instanceType: g4dn.xlarge`
- Add a GPU label to the node so workloads can target it:

```yaml
spec:
  template:
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/gpu-worker: ""
```

Apply it:

```bash
oc apply -f gpu-machineset.yaml
```

#### Install the NVIDIA GPU Operator

Before scaling up, install the NVIDIA GPU Operator from OperatorHub. It handles driver installation automatically via a DaemonSet that targets GPU nodes. Without this, your GPU node will come up but your app won't be able to use the GPU.

#### Scale up when you need the GPU, scale down when done

```bash
# Spin up the GPU node
oc scale machineset ocpcluster-gpu-us-east-1a -n openshift-machine-api --replicas=1

# Wait for node to be Ready
oc get nodes -w

# When finished, terminate the instance (MachineSet definition is preserved)
oc scale machineset ocpcluster-gpu-us-east-1a -n openshift-machine-api --replicas=0
```

Scaling to 0 is the better habit over deleting the MachineSet — you keep the definition and can scale back up any time without rebuilding the YAML.

---

### Summary of changes since previous posts

| What | Before | After |
|---|---|---|
| Control plane nodes | 3 | 1 |
| Worker nodes | 3 | 1 |
| Worker pricing | On-demand | Spot |
| Availability zones | 2 (us-east-1a, us-east-1b) | 1 (us-east-1a) |
| Subnets | 4 | 2 |
| Credentials mode | Mint (default) | Passthrough |
| Subnet tagging | Not required | Required |
| GPU nodes | N/A | On-demand via MachineSet scaling |

The result is a cluster that costs a fraction of the original setup to run, handles GPU testing without ongoing GPU spend, and works within the tighter IAM restrictions my org has put in place.
