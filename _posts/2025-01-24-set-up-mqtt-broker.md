---
layout: single
title:  "Quickly set up MQTT broker and clients"
categories: [mqtt]
tags: [mqtt]
excerpt: "Quickly set up a MQTT broker when you need one" #this is a custom variable meant for a short description to be displayed on home page
toc: true
---

<figure>
    <a href="/assets/mqtt-broker/mosquitto-logo.png"><img src="/assets/mqtt-broker/mosquitto-logo.png"></a>
</figure>

First post of 2025! I've been working on a few things but nothing that was worthy of a new post. Frustratingly, I've made slow progress with a challenging MQTT use case so I'm going to break this down into a few easy posts that I can refer to in future.

### How to quickly deploy an MQTT broker

#### Start with Ubuntu
An MQTT broker can run on Windows, Linux, and of course can run inside K8s. I will document a quick K8s set up in a future post, but for now we'll start with Ubuntu 22.04 LTS

#### Choose and install MQTT software

I'm going to choose [Mosquitto](https://mosquitto.org/) because it's popular. HiveMQ is also popular and perhaps I'll do that in future too.

Mosquitto is available in the default package repositories, so you could run ```sudo apt install -y mosquitto``` and I will show an example of that below.

But after reading instructions from a few sites, you can also add the [mosquitto-dev PPA](https://launchpad.net/~mosquitto-dev/+archive/ubuntu/mosquitto-ppa) to your repositories list to get more recent versions of mosquitto. So let's do that.

At command line of Ubuntu VM:

{% highlight bash %}
sudo apt-get update
sudo apt-get install curl gnupg2 wget git apt-transport-https ca-certificates -y #install pre-reqs
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa -y #add repo 
sudo apt-get update
sudo apt-get install mosquitto -y
sudo apt-get install mosquitto mosquitto-clients -y #run this if you also want to install client
#sudo systemctl status mosquitto #verify installation successful
#mosquitto -v #see version installed
{% endhighlight %}

Alternatively, you could install from default repositories. As an example only, today (Jan 24, 2025) when I deploy a fresh Azure VM with Ubuntu 22.04, if I run the script above I will see that mosquitto version 2.0.20 is running. If I run the script below, it's mosquitto version 2.0.11.

{% highlight bash %}
sudo apt update 
sudo apt install -y mosquitto
#sudo systemctl status mosquitto #verify installation successful
#mosquitto -v #see version installed
{% endhighlight %}

So, I'm going to run with the first script, the one that installed 2.0.20

#### Configure mosquitto

You will notice there is now a file at ```/etc/mosquitto/mosquitto.conf```, and there is a directory at ```/etc/mosquitto/conf.d/```. You can edit the .conf file, and/or create your own .conf files in the directory to configure mosquitto.

First, let's set a password. A file will be created at `/etc/mosquitto/passwd` where the username is **mqttuser** and the password is hashed in that file. We also need to set the correct ownership of that password file.

Second, let's create a file at `/etc/mosquitto/conf.d/default.conf`. We'll add a listener on port 1883 and tell mosquitto about the password file.

Lastly we will restart the mosquitto service.

{% highlight bash %}

sudo mosquitto_passwd -c /etc/mosquitto/passwd mqttuser
#enter password here

sudo chown mosquitto:mosquitto /etc/mosquitto/passwd

sudo bash -c 'cat > /etc/mosquitto/conf.d/default.conf <<EOF
listener 1883
password_file /etc/mosquitto/passwd
EOF
'
sudo systemctl restart mosquitto
{% endhighlight %}

#### Test connectivity

##### Locally on the same VM

For this you must have the mqtt clients installed, so I have just run `sudo apt-get install mosquitto-clients -y` on my Ubuntu VM and will test locally on the machine.

Create a client to subscribe:

{% highlight bash %}
mosquitto_sub -u mqttuser -P password -v -t "hello/topic"
{% endhighlight %}

Now, create a client to publish:

{% highlight bash %}
mosquitto_pub -u mqttuser -P password -t 'hello/topic' -m 'hello MQTT'
{% endhighlight %}

##### Over the network using a different MQTT client software

In this case I have set up a MQTT client on another VM, using the [MQTT CLI](https://hivemq.github.io/mqtt-cli) from HiveMQ. I'll include the commands to install this MQTT client cli purely for the sake of trying something different.

Notice that I include `-h x.x.x.x` in my command - that's the public IP address of the temporary MQTT broker I have set up.

{% highlight bash %}
#install mqtt cli
wget https://github.com/hivemq/mqtt-cli/releases/download/v4.35.0/mqtt-cli-4.35.0.deb
sudo apt install ./mqtt-cli-4.35.0.deb

mqtt sub -t hello/topic -h x.x.x.x -u mqttuser -pw password
{% endhighlight %}

Then, in another window on the same box:

{% highlight bash %}
mqtt pub -t hello/topic -m '12345' -h x.x.x.x -u mqttuser -pw password
{% endhighlight %}


