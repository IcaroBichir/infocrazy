---
layout: post
title: Configure Nginx to resolve DNS during the application uptime
modified:
categories: posts 
excerpt: "Making nginx smarter when the AWS ELB changes the Public IP"
tags: [nginx, AWS, ELB, ElasticLoadBalancer, webserver, proxy]
comments: true
image:
  feature: sample-image-11.jpg
  credit: TreeCanada
  creditlink: https://treecanada.ca/en/
date: 2016-02-26T00:04:09-03:00
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>General Vision</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

> I've assumed that you already have nginx installed on your system, if not <strike>don't worry</strike>, you can access the <a href="https://www.nginx.com/resources/wiki/start/topics/tutorials/install" target="_blank">nginx official repository</a> and install it.

### How it works

Nginx is a amazing web server and reverse proxy. It can be configured in different million ways and great things can be done. It's often used side by side with another server's, like Guicorn or Tomcat. Nginx allows buffered and queued clients, generating more speed to the application worker process.

This is the default configuration for nginx reverse proxy, using ``proxy_pass`` to redirect the requests on port 8080:

{% highlight bash %}
server {
    proxy_pass http://localhost:8080
}
{% endhighlight %}

This simple and reliable configuration works on almost every environment, servers and applications. It's a 1-1 connection between the nginx web server and the application server. When you identify any error on this, probably there is some problem with your backend. <strike>I'm sorry about that, fella</strike>

### Life ain't all sunshine and rainbows

When we use <a href="https://aws.amazon.com/elasticloadbalancing" target="_blank">AWS ELB</a>, it works on 90% of the time, because elastic load balancers have the habit to change their IP address and sometimes, when this occurs, <strike>run to the hills</strike> the request continuous to resolve the "old" IP address, on the default configuration of upstream and proxy modules the nginx never resolve DNS on runtime, just at startup, causing the service failure, because the backend continues sending the requests to the dropped IP.

### Configure to "re-resolve" DNS

There is a way to force nginx to re-resolve DNS during the application uptime <strike>Thankfully</strike>, using <a href="http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver">resolver</a>, <a href="http://nginx.org/en/docs/http/ngx_http_proxy_module.html">proxy_pass</a>, <a href="http://nginx.org/en/docs/http/ngx_http_upstream_module.html" target="_blank">upstream</a> feature and <a href="https://en.wikipedia.org/wiki/Regular_expression" target="_blank">regular expressions</a> you can force nginx to check if the DNS is working. If not, it loads the new IP. 

{% highlight bash %}
resolver 10.10.10.2;

server {
  set $upstream_endpoint your_elb_address_here.us-east-1.elb.amazonaws.com;
  
  location /api {
    rewrite ˆ/api(.*) /$1 break;
    proxy_pass http://$upstream_endpoint/api;
  }

  location /website {
    rewrite ˆ/website(.*) /$1 break;
    proxy_pass http://$upstream_endpoint/website;
  }
}
{% endhighlight %}

Nginx >= 1.1.9 will re-resolve DNS records based on their TTL, but with this <strike>little</strike> configuration, your nginx web server will be able to override the DNS records.

### “End Of Line” – The MCP, TRON

If you have some question or update about this procedure, please contact me.





