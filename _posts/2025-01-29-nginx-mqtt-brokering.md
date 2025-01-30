---
layout: single
title:  "Using NGINX Plus to apply security to MQTT connections"
categories: [mqtt]
tags: [mqtt, nginx]
excerpt: "This post ties together a few previous posts and shows how to use NGINX to secure MQTT connections" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
If you search for guides and documentation on using NGINX to secure MQTT messages, you will most likely find guides where:
- NGINX is running on a standalone instance (not in K8s)
  - if you are using K8s, you'll find guides to deploy NGINX Ingress Controller (not NGINX Plus running as a typical dataplane proxy)
- NGINX is decrypting TLS for a MQTT broker

In my case here, I need:
- NGINX to run inside K8s
- I will prefer NGINX Plus as a straightforward data plane installation (not the Ingress Controller image)
- I want NGINX to apply TLS and mTLS to the backend connection (not decrypt the client-side connection, but encrypt the server-side connection)
- I want NGINX to choose it's upstream based on the incoming CONNECT message.

### Overview of my target architecture
<figure>
    <a href="/assets/mqtt-broker/MQTT PoC - NGINX-security.png"><img src="/assets/mqtt-broker/MQTT PoC - NGINX-security.png"></a>
    <figcaption>Target architecture</figcaption>
</figure>

### Deploying my MQTT client and broker

#### Client in K8s (left side of diagram)
Because I am building a quick demo, I just followed my own instructions from [my recent post]({% post_url 2025-01-25-set-up-mqtt-in-k8s %}) and created an installation of Mosquitto inside K8s. Nothing really special here. I don't really need a broker, I really just want the command line tools but this was a quick way of doing this.

#### Server outside of K8s (right side of diagram)
Again I followed my own instructions from a [different previous post]{% post_url 2025-01-24-set-up-mqtt-broker %}. I built an Ubuntu VM and installed Mosquitto. But unlike last time, I configured a few more settings:
- I **only** allowed port 8883 from Internet to my VM (note: port 1883 denied)
- This installation listens on 8883 and uses TLS for secure transport
- This installation requires a client certificate from MQTT clients (ie, mTLS)

I used a self-signed cert/key pair for the TLS listener on the MQTT broker. So my file at `/etc/mosquitto/conf.d/default.conf` looks like this: 

{% highlight bash %}
#MQTT (secured with TLS)
listener 8883 0.0.0.0 #listen on secure port
require_certificate true #require mTLS
cafile /etc/mosquitto/certs/tls.crt # the same self-signed cert/key pair is used for mTLS, so I have used the cert here as the CA certs file
keyfile /etc/mosquitto/certs/tls.key #keyfile for TLS
certfile /etc/mosquitto/certs/tls.crt #certfile for TLS
password_file /etc/mosquitto/passwd
{% endhighlight %}

### Deploying NGINX Plus to proxy MQTT to MQTT(S)
Now you can see we have a client that uses regular MQTT and no username/password and a server that requires MQTT(S) as well as mTLS and username/password. We need NGINX Plus to understand the MQTT protocol and insert this security for us.

#### Running NGINX Plus inside K8s
In this case I am not installing the NGINX Plus Ingress Controller, but regular NGINX Plus running in a container. I want to mount a regular NGINX config file and I don't need to monitor NGINX services to configure upstream servers.

I'll include a link to all my YAML manifests I used to do this, but here's the deployment that should make things straightforward. *Note: You need to use NGINX Plus (not open source) if you want NGINX to natively understand MQTT and be able to preread or edit MQTT values*

{% highlight bash %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      automountServiceAccountToken: true
      containers:
      - env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        image: private-registry.nginx.com/nginx-plus/base:r33 #this is a private repo that requires authentication. The regcred secret contains a JWT that customers can download from myF5.
        imagePullPolicy: IfNotPresent
        name: nginx-plus
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 1883
          name: mqtt-port
          protocol: TCP
        - containerPort: 8081
          name: readiness-port
          protocol: TCP
        - containerPort: 9113
          name: prometheus
          protocol: TCP
        - containerPort: 9114
          name: service-insight
          protocol: TCP

        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: "/etc/nginx/license.jwt"
          subPath: license.jwt
          name: license-token
        - mountPath: "/etc/nginx/nginx.conf"
          subPath: nginx.conf
          name: nginx-config-file
        - mountPath: "/etc/nginx/tls.crt"
          subPath: tls.crt
          name: mqtt-cert
        - mountPath: "/etc/nginx/tls.key"
          subPath: tls.key
          name: mqtt-key
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: regcred #The regcred secret contains a JWT that customers can download from myF5.
      restartPolicy: Always
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      serviceAccount: nginx
      serviceAccountName: nginx
      terminationGracePeriodSeconds: 30
      volumes:
      - name: license-token
        secret:
          secretName: license-token
      - name: nginx-config-file
        configMap:
          name: nginx-config-file
      - name: mqtt-cert
        secret:
          secretName: mqtt-cert-key-secret
      - name: mqtt-key
        secret:
          secretName: mqtt-cert-key-secret
{% endhighlight %}

Here are the files I used to deploy the NGINX Plus service in K8s.
- [cert-key-secret.yaml](/assets/mqtt-broker/yaml-manifests/nginx/cert-key-secret.yaml){:target="_blank"} *#secret removed. This is just a self-signed TLS cert.*
- [deployment.yaml](/assets/mqtt-broker/yaml-manifests/nginx/deployment.yaml){:target="_blank"}
- [license-token.yaml](/assets/mqtt-broker/yaml-manifests/nginx/license-token.yaml){:target="_blank"} *#secret removed. This is the license that must exist on disk at /etc/nginx/license.jwt*
- [nginx-config-file.yaml](/assets/mqtt-broker/yaml-manifests/nginx/nginx-config-file.yaml){:target="_blank"}
- [ns.yaml](/assets/mqtt-broker/yaml-manifests/nginx/ns.yaml){:target="_blank"}
- [regcred.yaml](/assets/mqtt-broker/yaml-manifests/nginx/regcred.yaml){:target="_blank"} *#secret removed. This is used to pull the image from F5's registry.*
- [sa.yaml](/assets/mqtt-broker/yaml-manifests/nginx/sa.yaml){:target="_blank"}
- [service.yaml](/assets/mqtt-broker/yaml-manifests/nginx/service.yaml){:target="_blank"}

#### Configuring NGINX Plus
Let's take a look at the NGINX config file, as that's what applies the security in our diagram.

- Lines 9 & 12 shows that NGINX listens on tcp/1883 but sends to upstream on tcp/8883
- line 13 shows that we are using TLS for backend connection
- lines 18-19 are for mTLS, so that NGINX sends a client cert for backend connection
- lines 5, 25 & 26 are required if you want to insert a username and password into the MQTT CONNECT message.

```
....<removed some text above>....
    stream {
        resolver 8.8.8.8 valid=10s;
        mqtt on;
        mqtt_preread on;
        
        # Server block for MQTT handling
        server {
            listen 1883;  # MQTT default port

            # Proxy the connection to the appropriate upstream
            proxy_pass $mqtt_preread_clientid:8883;
            proxy_ssl on;
            proxy_ssl_server_name on;
            proxy_ssl_verify off;
            
            # Configure client certificate for mTLS
            proxy_ssl_certificate /etc/nginx/tls.crt;
            proxy_ssl_certificate_key /etc/nginx/tls.key;
            
            # Insert username and password in the CONNECT message
            set $username 'mqttuser';
            set $password 'ilovef5';
            
            mqtt_set_connect username $username;
            mqtt_set_connect password $password;
        }
        
        # Debug: Log Client Identifier
        log_format mqtt_log '$remote_addr [$time_local] $protocol '
                            'client_id=$mqtt_preread_clientid bytes_sent=$bytes_sent bytes_received=$bytes_received session_time=$session_time';
        access_log /var/log/nginx/mqtt_access.log mqtt_log;
    }
....<removed some text below>....
```