---
title:  "Running HAProxy in front of Kubernetes"
layout: post
date:   2015-07-26 14:28
description: Use HAProxy as a highly-available load balancer for your Kubernetes services
tags:
    - haproxy
    - kubernetes
    - high-availability
---

> "[HAProxy][haproxy] is a free, very fast and reliable solution offering high availability, load balancing, and proxying for TCP and HTTP-based applications. It is particularly suited for very high traffic web sites and powers quite a number of the world's most visited ones."

We're going to set up and use HAProxy to act as a load balancer in front of a Kubernetes cluster. HAProxy can proxy TCP and HTTP (although unfortunately not UDP) and optionally also provide SSL/TLS encryption for your HTTP backends. HAProxy can also use SNI (_Server Name Indication_) to load balance multiple (encrypted) sites, on a single IP address - useful in the increasingly scarce IPv4 world.

---

## Setup

We're going to be using HAProxy >= 1.5 in this post, as it includes support for SSL/TLS and SNI out of the box. The Debian 8 repositories as of today publish version 1.5.8 of HAProxy, so we'll be using that.

{% highlight bash linenos %}
apt-get update
apt-get install haproxy
{% endhighlight %}

The next sections shall explain configuring HAProxy through its `/etc/haproxy/haproxy.cfg` file.

---

## Backends

In HAProxy we must define our 'backends', which are where HAProxy can connect to your internal Kubernetes services. Once you've exposed your Kubernetes services (through a service definition file) you should have a port on each Kubernetes node that should itself proxy and balance across your cluster. So, HAProxy will connect to any of the nodes in your cluster on the specified port for that particular service, and it will be routed through your cluster by Kubernetes itself to your pods.

So, say our service has been exposed on port 32000 of each node in the cluster - we must now instruct HAProxy to _connect_ to any of the nodes in the cluster on port 32000.

{% highlight nginx linenos %}
backend james-munnelly-eu
    mode http
    balance leastconn
    option forwardfor
    cookie SRV_ID prefix

    server node1 10.20.40.60:32000 cookie check
    server node2 10.20.40.61:32000 cookie check
    server node3 10.20.40.61:32000 cookie check
{% endhighlight %}

Here, you can see we've defined 3 nodes that are each serving on port 32000. We also define a SRV_ID cookie, in order to stick sessions from a particular client, to a particular backend server (sticky sessions).

---

## Frontends

Below is an example frontend for a basic HTTP virtual host configuration. A switch is performed on the server name and the appropriate backend is then chosen.

{% highlight nginx linenos %}
frontend http-in
    bind :80
    type http

    reqadd X-Forwarded-Proto:\ http

    use_backend marley-landing if { hdr(host) -i marley.xyz }
    use_backend james-munnelly-eu if { hdr(host) -i james.munnelly.eu }
    use_backend kube-ui if { hdr(host) -i manage.marley.xyz }

    default_backend marley-landing
{% endhighlight %}

First, we name our frontend `http-in`. This is just a familar name for us to remember what the frontend is for. We bind to port 80 on the load balancer with `bind :80` and set the type to `http`. This allows HAProxy to set HTTP headers, including the X-Forwarded-Proto header (and any others you may want to add).

We then do a check on the field `hdr(host)`, which is the hostname the client is accessing this frontend on. This is derived either from HTTP headers, or SNI. This field dictates which backend service is chosen. If no backend matches, the default `marley-landing` backend (not shown here) is chosen.

---
#### SSL with _Server Name Indication_

As an added bonus, HAProxy can also provide SSL termination for your services and encrypt traffic to the user. Here's an example frontend configuration for this:

{% highlight nginx linenos %}
frontend https-in
    bind :443 ssl crt /var/lib/haproxy/ssl/certs.d

    reqadd X-Forwarded-Proto:\ https

    use_backend marley-landing if { hdr(host) -i marley.xyz }
    use_backend james-munnelly-eu if { hdr(host) -i james.munnelly.eu }
    use_backend kube-ui if { hdr(host) -i manage.marley.xyz }

    default_backend marley-landing
{% endhighlight %}

There's only a few very small diffierences in this configuration. We (obviously) must change the name of our frontend, here I've chosen `https-in` to be descriptive as to what this frontend provides. We bind to port 443, the default HTTPS port, and also provide some extra parameters: `ssl crt /var/lib/haproxy/ssl/certs.d`. This instructs HAProxy to enable SSL, and find certificates in the `/var/lib/haproxy/ssl/certs.d` folder which should contain a set a pem encoded public/private keypairs named `www.example.com.pem`.

We also adjust the `X-Forwarded-Proto` header to reflect the HTTP scheme by setting it to `https`.

---
## Conclusion

This should get you up and running with HAProxy, in an albeit manual configuration. I may in future look to writing an auto-configuration tool for HAProxy, that'd monitor the Kubernetes API for new services and automatically setup a new HAProxy configuration to load balance it.

[haproxy]: http://www.haproxy.org/