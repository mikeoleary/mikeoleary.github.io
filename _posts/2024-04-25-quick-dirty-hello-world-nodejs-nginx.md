---
layout: single
title:  "Source IP / PROXY Protocol demo using NGINX, NodeJS"
categories: [nginx]
tags: [nginx, nodejs]
excerpt: "This started as a quick and dirty demo to show NGINX and Proxy Protocol, and turned into a Node JS web app that I'll probably re-use." #this is a custom variable meant for a short description to be displayed on home page
---
### Customer requirements:

>1. Running in Google Cloud
2. We want a **global** load balancer. I.e, we want a public IP address that is advertised with Anycast IP to the Internet.
3. We want to know **true source IP** address of clients when they reach our web server.
4. We do **not** want to use an Application Load Balancer
    - i.e., we *do not want to perform TLS decryption* within a Google LB. 
    - Therefore *we cannot use X-Forwarded-For* headers to maintain true source IP
5. Additionally, we use Cloud Armor, so let us know how we can add on a CDN/DDoS/WAF provider

### Let's start with Google Load Balancers

If we start with [this guide](https://cloud.google.com/load-balancing/docs/choosing-load-balancer) from Google's documentation, we know that we need to end up with a **global, TCP-only** LB. That means we'll end up on the highlighted LB type of "Global external proxy Network Load Balancer".

<figure>
    <a href="/assets/gcp-tcp-global-lb/lb-product-tree-annotated.png"><img src="/assets/gcp-tcp-global-lb/lb-product-tree-annotated.png"></a>
    <figcaption>Choosing a Google Load Balancer.</figcaption>
</figure>

#### But wait, there's a problem!

Notice, this option that we are led to requires us to **proxy** the traffic, not preserving source IP address as we would with **passthrough** types. This is because "Global" LB's, where your IP address is advertised using Anycast, all require *proxying* (an F5 customer might call this 'SNATing') of the traffic to the EC2 instance in a given region, in order to maintain symmetry. After all, the incoming traffic could be coming from any of Google's globally-distributed [GFE's](https://cloud.google.com/docs/security/infrastructure/design#google-frontend-service), so it's understandable that proxying is required.

#### Proxy Protocol saves the day

I'll just copy/paste from the [relevant section in the documentation](https://cloud.google.com/load-balancing/docs/tcp#target-proxies) specific to our chosen LB type:

>By default, the target proxy does not preserve the original client IP address and port information. You can preserve this information by enabling the PROXY protocol on the target proxy.

<figure>
    <a href="/assets/gcp-tcp-global-lb/gcp-lb-venn-diagram.png"><img src="/assets/gcp-tcp-global-lb/gcp-lb-venn-diagram.png"></a>
    <figcaption>You can have all 3 of these things, but you must use Proxy Protocol to achieve it.</figcaption>
</figure>

### Implementing Proxy Protocol using NGINX
**Assuming you have already configured your GCP LB to append Proxy Protcol headers**, you can configure NGINX to expect Proxy protocol. Then, you can configure NGINX to append an additional header when proxying traffic with the value of the source IP obtained from Proxy Protocol:

```
server {
  listen 80 proxy_protocol; # this line tells NGINX to expect traffic with proxy protocol
  server_name customer1.my-f5.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header x-nginx-ip $server_addr; # this is the header I added which will store the IP address of the NGINX server
    proxy_set_header x-proxy-protocol-source-ip $proxy_protocol_addr; # this is the header I added to show the src IP address obtained from Proxy protocol.
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr; # this is the header I added to show the src IP of the connection between Google's front end and NGINX.
    proxy_cache_bypass $http_upgrade;
  }
}
```

You might notice that NGINX is simply proxying to <code>http://localhost:3000</code>. What is that? A simple Node JS app that will display a page with these headers:


```js
const express = require('express');
const app = express();
const port = 3000;

// set the view engine to ejs
app.set('view engine', 'ejs');

app.get('/', (req, res) => {
        const proxy_protocol_addr = req.headers['x-proxy-protocol-source-ip'];
        const source_ip_addr = req.headers['x-real-ip'];
        const array_headers = JSON.stringify(req.headers, null, 2);
        const nginx_ip_addr = req.headers['x-nginx-ip'];
    res.render('index', {
        proxy_protocol_addr: proxy_protocol_addr,
        source_ip_addr: source_ip_addr,
        array_headers: array_headers,
        nginx_ip_addr: nginx_ip_addr
    });
})
app.listen(port, () => {
  console.log('Server is listenting on port 3000');
})

```

Oh, and this server uses EJS view engine, so the file `views/index.ejs` is here:

````html
<!DOCTYPE html>
<html lang="en">
<head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale-1">
        <title>Demo App</title>
</head>
<body class="container">
<main>
        <h2>Hello World!</h2>
        <p>True source IP (the value of <code>$proxy_protocol_addr</code>) is <b><%= typeof proxy_protocol_addr != 'undefined' ? proxy_protocol_addr : '' %></b></p>
        <p>IP address that NGINX recieved the connection from (the value of <code>$remote_addr</code>) is <b><%= typeof source_ip_addr != 'undefined' ? source_ip_addr : '' %> </b></p>
        <p>IP address that NGINX is running on (the value of <code>$server_addr</code>) is <b><%= typeof nginx_ip_addr != 'undefined' ? nginx_ip_addr : '' %></b><p>
        <h3>Request Headers at the app:</h3>
        <pre><%= typeof array_headers != 'undefined' ? array_headers : '' %></pre>
</main>
</body>
</html>
````

### Cloud Armor (Google's DDoS / WAF provider)

This is an [easy add-on](https://cloud.google.com/blog/products/identity-security/cloud-armor-adds-more-edge-security-policies-proxy-load-balancers) when using Google Load Balancers. 
>Customers can use these proxy endpoints to leverage the Google global load balancing infrastructure to serve TCP-based workloads

### The end result
In the end, this is a small demo app to show true source IP using Global TCP Network Load Balancer, Proxy Protocol, NGINX, and Node JS. After setting this up, I realized I could have done this natively with NodeJS only and left NGINX out.

<figure>
    <a href="/assets/gcp-tcp-global-lb/demo-app-src-ip-nodejs.png"><img src="/assets/gcp-tcp-global-lb/demo-app-src-ip-nodejs.png"></a>
    <figcaption>Simple demo app showing src IP from Proxy Protocol</figcaption>
</figure>


