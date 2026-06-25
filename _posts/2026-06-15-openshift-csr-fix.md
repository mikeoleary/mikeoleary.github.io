---
title: "OpenShift won't come back after a 5-day stop? Check your CSRs (recursively)"
date: 2026-06-15
categories: [openshift]
tags: [openshift, kubernetes, aws]
excerpt: "My lab cluster sat stopped on EC2 for 5 days. oc get po came back full of memcache errors and nothing made sense until I found the actual culprit: kubelet serving certs, and a CSR approval queue that needed several passes."
toc: true
---

<figure>
    <a href="/assets/openshift-csr-fix/openshift-csr-fix-header.jpg"><img src="/assets/openshift-csr-fix/openshift-csr-fix-header.jpg"></a>
    <figcaption>Heavy machinery operator. Not the same thing.</figcaption>
</figure>


Quick one for future me.
 
I had my OpenShift lab (running on EC2, two nodes, AWS IPI) stopped for about 5 days in a row. Powered the instances back on, waited 10 minutes, ran `oc get po`, and got a wall of this:
 
```
E0625 10:08:04.938687   19482 memcache.go:287] couldn't get resource list for route.openshift.io/v1: the server is currently unable to handle the request
E0625 10:08:04.938692   19482 memcache.go:287] couldn't get resource list for user.openshift.io/v1: the server is currently unable to handle the request
E0625 10:08:04.939031   19482 memcache.go:287] couldn't get resource list for authorization.openshift.io/v1: the server is currently unable to handle the request
E0625 10:08:04.944833   19482 memcache.go:287] couldn't get resource list for oauth.openshift.io/v1: the server is currently unable to handle the request
E0625 10:08:04.945588   19482 memcache.go:287] couldn't get resource list for template.openshift.io/v1: the server is currently unable to handle the request
E0625 10:08:04.945938   19482 memcache.go:287] couldn't get resource list for image.openshift.io/v1: the server is currently unable to handle the request
```
 
Every single OpenShift-specific API group (`route`, `user`, `oauth`, `template`, `image`, `authorization`, `quota`, `project`, `build`, `security`, `apps`) erroring out, but `oc get nodes` worked fine and both nodes showed `Ready`. So the cluster was *alive*, just not all the way there. I used Claude to help troubleshoot obviously, but here's some notes for next time.
 
## First checkpoint: `oc get co`
 
This is always step one when something's off post-boot. Every operator came back:
 
```
NAME                  VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
...
openshift-apiserver   4.21.6    True        False         False      2d19h
authentication        4.21.6    True        False         False      2d
etcd                  4.21.6    True        False         False      2d19h
```
 
Available, not progressing, not degraded. Everything green. Which is a little maddening — the operator status said "fine" while the actual API was throwing `ServiceUnavailable`. Lesson here: ClusterOperator status reflects the operator's view of its own rollout, not whether the live request path between kube-apiserver and the aggregated API actually works right now.
 
## Narrowing it down
 
`oc get --raw /healthz` returned `ok`, so kube-apiserver itself was healthy. But:
 
```
$ oc get --raw /apis/route.openshift.io/v1
Error from server (ServiceUnavailable): the server is currently unable to handle the request
```
 
That's the aggregated API layer (openshift-apiserver) specifically. Checked events in that namespace — all 2 days stale, useless for "right now" debugging since events expire/rotate out. Went looking for logs instead and got the real clue:
 
```
$ oc logs -n openshift-apiserver -l apiserver=true --tail=100
Error from server: Get "https://10.0.2.198:10250/containerLogs/...": remote error: tls: internal error
```
 
kube-apiserver couldn't even open a TLS connection to the kubelet on port 10250 to fetch logs. Tried it again against an unrelated pod on the other node, same error, different IP:
 
```
$ kubectl logs -n kube-system cis1-f5-bigip-ctlr-787fc86-xc97n
Error from server: Get "https://10.0.2.241:10250/containerLogs/...": remote error: tls: internal error
```
 
Both nodes. Not node-specific. That ruled out a one-off hardware/cert fluke and pointed straight at kubelet serving certs across the whole cluster.
 
## The actual fix: approve pending CSRs (more than once)
 
```
oc get csr | grep Pending
```
 
Sure enough — pending CSRs. Approved them:
 
```
oc get csr -o name | xargs oc adm certificate approve
```
 
Re-tested. Still broken. Checked again, found *more* pending CSRs, approved those too. Had to repeat this three times total before everything cleared.
 
This makes sense once you think about what's happening on a fresh boot after days of being stopped: kubelet on each node comes up and requests a fresh serving cert. The cluster-machine-approver is supposed to auto-approve these, but it's an operator pod like everything else — if it isn't fully up yet when the first batch of CSRs lands, they sit pending. By the time you approve that batch, kubelet (or another component) generates the next round, and the approver still might not have caught up. Rinse and repeat until the queue actually drains.
 
The one-liner to just keep clearing it:
 
```
oc get csr -o name | xargs oc adm certificate approve
```
 
Run it, wait a few seconds, run `oc get csr | grep Pending` again, repeat until it comes back empty.
 
## Why this happens after a multi-day stop specifically
 
A short stop/start or simple reboot usually doesn't trigger this — kubelet's existing cert is still valid and the approver is healthy and quick. A 5-day stop seems to be enough that:
 
- Every node's kubelet needs new bootstrap/serving cert material on the way back up.
- The cluster-machine-approver itself needs to fully start before it can approve anything, so there's a window where CSRs pile up faster than they're cleared.
- That backlog doesn't necessarily clear in one pass — approving batch 1 can be exactly what unblocks the component that triggers batch 2.
I didn't see meaningful clock drift on either node (worth checking too, since `tls: internal error` can also mean a cert looks invalid due to time skew — `oc debug node/<name> -- chroot /host chronyc tracking` is the quick check), but in this case the CSR backlog was the whole story.
 
## tl;dr for next time
 
1. `oc get po` full of `memcache.go` errors for every OpenShift API group, but `oc get nodes` works → don't panic, check `oc get co` first.
2. All operators green but `oc get --raw /apis/<group>` still fails → check `oc logs` against any pod; if it fails with `tls: internal error` reaching the kubelet on `:10250`, that's your real signal.
3. `oc get csr | grep Pending` → approve with `oc get csr -o name | xargs oc adm certificate approve`.
4. **Check again.** And again. Don't assume one pass clears it after a multi-day stop — it took three rounds for me.