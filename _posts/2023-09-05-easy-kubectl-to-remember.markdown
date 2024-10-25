---
layout: single
title:  "kubectl commands to remember"
categories: kubernetes
tags: kubernetes
---

## Easy kubectl commands
This is a short list of kubectl commands that I use frequently. The idea of this blog post is to save me from searching my Notepad++ tabs looking for random notes I've taken in the last few weeks/months.

#### Quickly launch a troubleshooting pod 
Here is a command to give you a command line at a pod with curl installed:
```kubectl run -i --tty curl --image=curlimages/curl --restart=Never --rm -- sh```

Here is a command that will do a nslookup from within a pod in the cluster. Edit the DNS record as appropriate:
````bash
kubectl run -i dnsutils \
  --image=gcr.io/kubernetes-e2e-test-images/dnsutils:1.3 \
  --rm \
  -- nslookup backend-service.jobs.svc.cluster.local
````

#### Quickly enable autocompletion
{% highlight bash %}
source <(kubectl completion bash) # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
bash # reload bash shell
{% endhighlight %}



Otherwise use the cheat sheet from the docs: https://kubernetes.io/docs/reference/kubectl/cheatsheet/