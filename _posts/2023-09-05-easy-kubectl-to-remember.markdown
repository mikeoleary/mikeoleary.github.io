---
layout: single
title:  "kubectl commands to remember"
categories: kubernetes
---

## Easy kubectl commands
This is a short list of kubectl commands that I use frequently. The idea of this blog post is to save me from searching my Notepad++ tabs looking for random notes I've taken in the last few weeks/months.

#### Quickly launch a troubleshooting pod 
```kubectl run -i --tty curl --image=curlimages/curl --restart=Never -- sh```

#### Quickly enable autocompletion
{% highlight bash %}
source <(kubectl completion bash) # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
bash # reload bash shell
{% endhighlight %}



Otherwise use the cheat sheet from the docs: https://kubernetes.io/docs/reference/kubectl/cheatsheet/