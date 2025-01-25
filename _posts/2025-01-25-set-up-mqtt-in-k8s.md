---
layout: single
title:  "Setting up Mosquitto in K8s"
categories: [mqtt]
tags: [mqtt]
excerpt: "Deploy mosquitto in K8s when you need a broker or client tools in K8s quickly" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

Yesterday I posted about how to deploy a simple MQTT broker running Mosquitto on an Ubuntu VM. Today, we'll set up the same thing in K8s

### How to quickly deploy MQTT in K8s

#### Start with K8s
I like to deploy Azure (AKS) clusters, only because I'm pretty familiar with doing it quickly. For this simple demo, any distro should work.

#### Deploy these manifests
A namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mosquitto
```

A ConfigMap to create a config file that we will mount. Obviously we can change the settings in this file if we need.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: mosquitto
data:
  mosquitto.conf: |
    # DO NOT USE IN PRODUCTION
    allow_anonymous true
    # MQTT
    listener 1883
    protocol mqtt
```

Then a deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto
  namespace: mosquitto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mosquitto
  template:
    metadata:
      labels:
        app: mosquitto
    spec:
      containers:
      - image: eclipse-mosquitto
        imagePullPolicy: Always
        name: mosquitto
        ports:
        - containerPort: 8883
          protocol: TCP
        - containerPort: 1883
          protocol: TCP
        - containerPort: 9001
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /mosquitto/config/mosquitto.conf
          name: config
          subPath: mosquitto.conf
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      volumes:
      - configMap:
          name: mosquitto-config
        name: config
```

Then a service to expose it. I've used type LoadBalancer here but expose it however you need:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mosquitto-mqtt
  namespace: mosquitto
spec:
  ports:
  - name: mqtts
    port: 8883
    protocol: TCP
    targetPort: 8883
  - name: mqtt
    port: 1883
    protocol: TCP
    targetPort: 1883
  selector:
    app: mosquitto
  type: LoadBalancer
```

#### Test your deployment

Set up a subscriber:

```bash
kubectl exec -n mosquitto mosquitto-7968b59f9f-s29jg -it -- mosquitto_sub -t hello/topic
```

Then publish a message:

```bash
kubectl exec -n mosquitto mosquitto-7968b59f9f-s29jg -- mosquitto_pub -t hello/topic -m 123
```

### Resources
https://www.enabler.no/en/blog/mosquitto-mqtt-broker-in-kubernetes
