---
layout: post
title: Install and Configure Nginx With Naxsi
modified:
categories: 
excerpt: "Configure naxsi WAF in your nginx webserver"
tags: [nginx, naxsi, waf, security, webserver]
comments: true
image:
  feature: sample-image-3.jpeg
  credit: LegalSoft
  creditLink: http://legalsoft.info/
date: 2016-06-13T23:08:06-03:00
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

### With this article, you will have your webserver ready to production, filtering all requests with NAXSI WAF configured on nginx.

> Versions: nginx 1.8.1 + naxsi 1.5.3

> Tested on CentOS 7 and Ubuntu Trusty

> Execute all steps on root account

### Get the nginx and nasxi source code

### Compile and install from source nginx and naxsi

{% highlight bash %}
wget http://nginx.org/download/nginx-1.8.1.tar.gz
wget https://github.com/nbs-system/naxsi/archive/0.54.tar.gz
wget https://github.com/nbs-system/naxsi/archive/0.53.tar.gz
{% endhighlight %}

### Unpack the tar files

{% highlight bash %}
tar -xvzf nginx-1.8.1.tar.gz
tar -xvzf 0.54.tar.gz
tar -xvzf 0.53.tar.gz
{% endhighlight %}

### Remove old nginx files and move the naxsi directory

{% highlight bash %}
rm -rf /usr/local/nginx
mkdir /usr/local/nginx/
mv naxsi-0.54/  /usr/local/naxsi-0.54/
mv naxsi-0.53/nx_util/  /usr/local/naxsi-0.54/nx_util-0.53/
{% endhighlight %}

### Configure and compile the nginx on your linux kernel

{% highlight bash %}
cd nginx-1.8.1
./configure --conf-path=/usr/local/nginx/conf/nginx.conf \
--add-module=/usr/local/naxsi-0.54/naxsi_src/ \
--error-log-path=/var/log/nginx/error.log \
--http-client-body-temp-path=/usr/local/nginx/body \
--http-fastcgi-temp-path=/usr/local/nginx/fastcgi \
--http-uwsgi-temp-path=/usr/local/nginx/uwsgi \
--http-scgi-temp-path=/usr/local/nginx/scgi \
--http-log-path=/var/log/nginx/access.log \
--http-proxy-temp-path=/usr/local/nginx/proxy \
--lock-path=/var/run/nginx.lock \
--pid-path=/var/run/nginx.pid \
--with-http_ssl_module \
--with-http_ssl_module \
--with-http_addition_module \
--with-http_realip_module \
--with-http_gunzip_module \
--without-mail_pop3_module \
--without-mail_smtp_module \
--without-mail_imap_module \
--without-http_uwsgi_module \
--without-http_scgi_module \
--with-ipv6 \
--sbin-path=/usr/sbin/nginx \
--prefix=/usr/local/nginx
make
make install
{% endhighlight %}

### Copy naxsi base rules to nginx conf directory 

{% highlight bash %}
cp /usr/local/naxsi-0.54/naxsi_config/naxsi_core.rules /usr/local/nginx/conf/
{% endhighlight %}

### Edit the nginx.conf and include the naxsi base rules on http section

{% highlight bash %}
$ vim /usr/local/nginx/conf/nginx.conf

    http {
	    ...
        include				/usr/local/nginx/conf/naxsi_core.rules;
	    ...
    }
{% endhighlight %}

### Create your custom naxsi rules

{% highlight bash %}
$ touch /usr/local/nginx/conf/naxsi_custom.rules
$ cat << EOF >/usr/local/nginx/conf/naxsi_custom.rules
LearningMode;  
SecRulesEnabled;
DeniedUrl "/RequestDenied";

## check rules
CheckRule "$SQL >= 8" BLOCK;
CheckRule "$RFI >= 8" BLOCK;
CheckRule "$TRAVERSAL >= 4" BLOCK;
CheckRule "$EVADE >= 4" BLOCK;
CheckRule "$XSS >= 8" BLOCK;
EOF
{% endhighlight %}

### Include your custom rules on every server configuration

{% highlight bash %}
$ vim /usr/local/nginx/conf.d/*.conf
    server {
    	...
    	location / {
        	include			/usr/local/nginx/conf/naxsi_custom.rules;
        ...
    }
{% endhighlight %}

### Start your nginx, with naxsi compiled inside

{% highlight bash %}
$ sudo service nginx start
{% endhighlight %}

### After some days on LearningMode, configure the nx_util to create your custom whitelist

> ps: I used the 0.53 version, because the setup create the elasticsearch to execute the nx_util.py

{% highlight bash %}
$ cd /usr/local/naxsi-0.54/nx_util-0.53/
$ python setup.py build
$ python setup.py install
$ python nx_util.py -l /var/log/nginx/error.log -o >> /usr/local/nginx/conf/naxsi_custom.rules

{% endhighlight %}

### Look your configuration file and analyses the whitelist to understand the requests and his needs

### Turn off the naxsi LearningMode

{% highlight bash %}
sed -i 's/LearningMode/#LearningMode/' /usr/local/nginx/conf/naxsi.rules
{% endhighlight %}
