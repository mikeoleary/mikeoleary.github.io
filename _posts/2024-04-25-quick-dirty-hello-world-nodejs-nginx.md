---
layout: single
title:  "Google LB and Proxy Protocol demo using NGINX, NodeJS"
categories: [nginx]
tags: [nginx, nodejs]
excerpt: "This started as a quick and dirty demo to show NGINX and Proxy Protocol, and turned into a Node JS web app that I'll re-use one day." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
Recently I was talking to a customer who wanted to know what their options were in Google cloud for deploying devices behind load balancers with Google. This is a brief overview of their requirements, and a small demo.

### Customer requirements:

>1. We have VM's running applications in Google Cloud.
2. We want a **global** load balancer. 
    - i.e, we want a public IP address advertised via Anycast from each of Google's front end locations.
3. We want to know the **true source IP** of clients when they reach our web server.
4. We do **not** want to use an Application Load Balancer
    - i.e., we **do not** want to perform TLS decryption within a Google LB. 
    - therefore we **cannot** use X-Forwarded-For headers to preserve true source IP
5. Additionally, we'd like to use Cloud Armor. Please let us know if/how we can add on a CDN/DDoS/WAF provider.

### Let's start with Google Load Balancers

[This guide](https://cloud.google.com/load-balancing/docs/choosing-load-balancer) shows us several options for Google LB's. Because our requirements include **global, TCP-only** load balacing, we will choose the highlighted LB type of "Global external proxy Network Load Balancer".

<figure>
    <a href="/assets/gcp-tcp-global-lb/lb-product-tree-annotated.png"><img src="/assets/gcp-tcp-global-lb/lb-product-tree-annotated.png"></a>
    <figcaption>Choosing a Google Load Balancer.</figcaption>
</figure>

#### What about proxying vs passthrough?

Notice that global LB's **proxy** traffic. They do not preserve source IP address as a **passthrough** LB does. This is because global IP addresses are advertised  from globally-distributed [front end locations](https://cloud.google.com/docs/security/infrastructure/design#google-frontend-service). 

Proxying will maintain traffic symmetry, but Source NAT causes loss of the original client IP address. Fortunately, we can overcome this with Proxy protocol.

#### Proxy Protocol support with Google Load Balancers

The [documentation](https://cloud.google.com/load-balancing/docs/tcp#target-proxies) for TCP load balancers says it all:

>By default, the target proxy does not preserve the original client IP address and port information. You can preserve this information by enabling the PROXY protocol on the target proxy.

Without Proxy protocol support, we could only meet 2 of our core 3 requirements with a given solution. Proxy protocol allows us to meet all 3 simultaneously.
<figure>
    <a href="/assets/gcp-tcp-global-lb/gcp-lb-venn-diagram.png"><img src="/assets/gcp-tcp-global-lb/gcp-lb-venn-diagram.png"></a>
    <figcaption>You can have all 3 of these things, but you must use Proxy Protocol to achieve it.</figcaption>
</figure>

### Receiving Proxy Protocol using NGINX
Let's run NGINX on the VM to which our load balancer points. When proxying traffic, NGINX can append an additional header containing the value of the source IP obtained from Proxy Protocol:

```
server {
  listen 80 proxy_protocol; # tell NGINX to expect traffic with proxy protocol
  server_name customer1.my-f5.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header x-nginx-ip $server_addr; # append a header to pass the IP address of the NGINX server
    proxy_set_header x-proxy-protocol-source-ip $proxy_protocol_addr; # append a header to pass the src IP address obtained from Proxy protocol
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr; # append a header to pass the src IP of the connection between Google's front end LB and NGINX
    proxy_cache_bypass $http_upgrade;
  }
}
```

You might notice that NGINX is proxying to <code>http://localhost:3000</code>. I have a simple NodeJS app to display a page with these headers:

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

For completeness, NodeJS is using the EJS template engine to build our page. The file `views/index.ejs` is here:

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

### Cloud Armor

Cloud Armor is an [easy add-on](https://cloud.google.com/blog/products/identity-security/cloud-armor-adds-more-edge-security-policies-proxy-load-balancers) when using Google Load Balancers. Simply create a Cloud Armor security policy, add rules (for example, rate limiting) to this policy, and attach it to your TCP load balancer. In this way "edge protection" is applied to your Google workloads with very little effort.

### Our end result
This small demo app to shows that true source IP can be known to an application running inside Google, even when using the Global TCP Network Load Balancer. We've done this using Proxy protocol and NGINX, and we've used NodeJS simply to display a web page.

<figure>
    <a href="/assets/gcp-tcp-global-lb/demo-app-src-ip-nodejs.png"><img src="/assets/gcp-tcp-global-lb/demo-app-src-ip-nodejs.png"></a>
    <figcaption>Simple demo app showing src IP from Proxy Protocol</figcaption>
</figure>

Thanks for reading, and reach out with any questions!

