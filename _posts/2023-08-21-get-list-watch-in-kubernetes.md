---
layout: single
title:  "Get, List, Watch verbs in Kubernetes RBAC"
categories: [kubernetes]
tags: [kubernetes]
toc: true
excerpt: "This post describes the watch verb, which I had taken for granted before today" #this is a custom variable meant for a short description to be displayed on home page
---
I've been helping customers with F5 CIS for about 5 years now, but I learned something new today that I feel is relatively basic in retrospect: the difference between ```get```, ```list```, and ```watch``` in Kubernetes RBAC verbs.

### Kubernetes RBAC basics
To pass the CKA exam you should definitely understand the relationship between the objects of
- ```ServiceAccount```
- ```Role``` and ```RoleBinding```
- ```ClusterRole``` and ```ClusterRoleBinding```

Typically I deploy ingress controllers and in most day-to-day scenarios I deploy NGINX with a ServiceAccount and ClusterRole that looks like this:

{% highlight yaml %}
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nginx-ingress
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  verbs:
  - get
  - list
  - watch
### < redacted for brevity, but typically there's more API groups like secrets and given CRD's >
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nginx-ingress
subjects:
- kind: ServiceAccount
  name: nginx-ingress
  namespace: nginx-ingress
roleRef:
  kind: ClusterRole
  name: nginx-ingress
  apiGroup: rbac.authorization.k8s.io
{% endhighlight %}

As you can see from above, I'm granting the ServiceAccount the permissions required to ```get```, ```list```, and ```watch``` certain objects within Kubernetes.

### Kubernetes API request verbs
Requests to the K8s api have a request *verb*. Typically we see ```get``` or ```list``` in Roles, but is there an official list of all possible verbs? In searching for the answer I came across [this answer](https://stackoverflow.com/questions/57661494/list-of-kubernetes-rbac-rule-verbs) and the following command, which will list the complete list of possible verbs:

```bash
$ kubectl api-resources --no-headers --sort-by name -o wide | sed 's/.*\[//g' | tr -d "]" | tr " " "\n" | sort | uniq
create
delete
deletecollection
get
list
patch
update
watch
```

#### Which verb should I use?
I'll just reference the documentation here: [Determine the Request Verb](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb)

|HTTP verb| request verb|
|---|---|
|POST| create|
|GET, HEAD|	get (for individual resources), list (for collections, including full object content), watch (for watching an individual resource or collection of resources)|
|PUT|	update|
|PATCH|	patch|
|DELETE| delete (for individual resources), deletecollection (for collections)|

This part was interesting:

<p>The get, list and watch verbs can all return the full details of a resource.</p>{: .notice}

It's pretty easy to understand how `get` and `list` are different if you've made API calls before, but what is involved with `watch` ?

### How is watch special?
In this article I only want to focus on ```watch``` and how it's different from ```get``` and ```list```.

I have always known that CIS "subscribes" to the Kubernetes API to "watch" for changes to objects, but I naively just assumed that CIS and other operators made frequent API calls in the style of HTTP GET requests. I.e., I thought "watching" meant "frequent getting or listing".

Of course this would be inefficient, and today I started thinking about what "subscribe" really means. After searching for only a minute or two I read this nice sentence from a StackOverflow [answer](https://stackoverflow.com/a/58160692/8913988):

> watch is a special verb that gives you permission to see updates on resources in real time. 

Reading [another answer](https://stackoverflow.com/a/58165061/8913988), I see

> They open a streaming connection that returns you the full manifest of a Deployment whenever it changes (or when a new one is created).

### Efficient detection of changes
Now I get it! With words like "real time" and "stream" I realized I should find what I'm looking for in the official documentation, and of course it's right here: [Efficient detection of changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)

> When you send a watch request, the API server responds with a stream of changes.

You can read the documentation further for the sequence of API calls that need to be made, but here's a nice summary:
1. Here is an initial ```get``` for all pods in the test namespace. Notice the "resourceVersion" returned in line 9.
   ```bash
   GET /api/v1/namespaces/test/pods
   ---
   200 OK
   Content-Type: application/json

   {
     "kind": "PodList",
     "apiVersion": "v1",
     "metadata": {"resourceVersion":"10245"},
     "items": [...]
   }
   ```
2. Here is a followup request with `watch=1` and `resourceVersion` request parameters. Notice this response has `Transfer-Encoding: chunked` to indicate that this is a [long-running streaming connection](https://en.wikipedia.org/wiki/Chunked_transfer_encoding) and more data will be sent from server to client.
   ```bash
   GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
   ---
   200 OK
   Transfer-Encoding: chunked
   Content-Type: application/json

   {
     "type": "ADDED",
     "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
   }
   {
     "type": "MODIFIED",
     "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...}
   }
   ...
   ```

K8s client instrumentations like Go and Java handle this for developers, and that's something outside of my current concerns. For some other time, [here](https://kubernetes.io/docs/reference/#officially-supported-client-libraries) is a list of officially supported client libraries to call the Kubernetes API.

I'll quote another [nice article](https://www.baeldung.com/java-kubernetes-watch):

> However, passing watch=true changes its behavior dramatically:
  The response now consists of a series of modification events, containing the type of modification and the affected object.
  **The connection will be kept open after sending the initial batch of events, using a technique called long polling**

#### Resource Versions and Bookmarks
This article goes on to explain how to make watch API calls, and how **Resource Versions** and **Bookmarks** can be used to make this streaming more efficient, especially when long-lived streams are dropped and must be re-initiated.

- **Resource versions** allow a client to indicate to the server that they only want changes streamed after a given resource version, so the server doesn't send the client more historical events than necessary.
- **Bookmarks** allow a server to tell a client the most recent Resource Version it has sent. This way a client can cache this somewhere, and in the event that they have to re-initiate the watch stream, they can use the Resource Version from the last bookmark they were given by the server.

[Here](https://stackoverflow.com/a/69667446/8913988) is a nice answer to the question *What are bookmarks trying to solve?*

>Its intended to reduce the load on kube-apiserver from clients that issue Watch requests for a resource type. Bookmarks are intended to let the client know that the server has sent the client all events up to the resourceVersion specified in the Bookmark event.

### Summary
- The ```watch``` permission in Kubernetes RBAC is special. It allows clients to 'subscribe' to changes from the Kubernetes API server with long-lived connections
- This is more efficient than many frequent ```get``` or ```list``` requests
- To make ```watch``` more efficient, clients should use Resource Versions and Bookmarks
- All of this is instrumented by libraries used by developers. I don't think the average Kubernetes admin needs to understand much deeper than this, unless they are developing operators or other clients of the Kubernetes API.
