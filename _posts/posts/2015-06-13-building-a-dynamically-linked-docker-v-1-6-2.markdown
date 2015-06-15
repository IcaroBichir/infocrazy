---
layout: post
title: "Building a Dynamically Linked Docker v-1.6.2"
modified: 2015-06-13
categories: posts
excerpt: "Build docker dinamically to solve problems with Udev and devicemapper"
tags: [docker, dynamic, static, error, solved]
comments: true
image:
  feature: sample-image-8.jpg
  credit: Docker
  creditlink: http://www.docker.com/ 
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Visão Geral</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

If you are having problem with Docker, receiving this following error when initiate a new docker:

{% highlight bash %}
time="2015-06-10T17:24:26-03:00" level=fatal msg="Error response from daemon: 
Cannot start container 16e1b3fabf3c9d9c9383f9ab59fef558417bb61020dbbb19b947540af6999b78:
Error getting container 16e1b3fabf3c9d9c9383f9ab59fef558417bb61020dbbb19b947540af6999b78 from driver devicemapper: 
Error mounting '/dev/mapper/docker-202:1-531040-16e1b3fabf3c9d9c9383f9ab59fef558417bb61020dbbb19b947540af6999b78' 
on '/var/lib/docker/devicemapper/mnt/16e1b3fabf3c9d9c9383f9ab59fef558417bb61020dbbb19b947540af6999b78': no such file or directory" 
{% endhighlight %}

Check the Docker and Udev syncronization.

{% highlight bash %}
$ docker info | grep Udev
Udev Sync Supported: false
{% endhighlight %}

Docker is meant to be built statically as it tries to contain everything it needs in the binary. If the return is false, your Docker is running statically and we <a href="https://github.com/docker/docker/issues/4036#issuecomment-111174341"> need to work on that.</a> 

Check if docker is running dynamically with the following command:

{% highlight bash %}
$ ldd /usr/bin/docker
not a dynamic executable
{% endhighlight %}

Catch all the versions of Docker running in your system and remove then.

{% highlight bash %}
$ dpkg-query -l *docker*
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                              Version               Architecture          Description
+++-=================================-=====================-=====================-=======================================================================
ii  docker                            1.5-1                 amd64                 System tray for KDE3/GNOME2 docklet applications

$ sudo apt-get remove docker
$ sudo apt-get autoremove
{% endhighlight %}

###Dependencies 

(running ubuntu-trusty-64 14.04.2 LTS)

Let’s begin by installing the regular Docker package:

{% highlight bash %}
sudo apt-get install -qy apt-transport-https
sudo apt-get install golang build-essential libdevmapper-dev golang-gosqlite-dev
sudo apt-get install uuid-dev libattr1-dev zlib1g-dev libacl1-dev e2fslibs-dev libblkid-dev liblzo2-dev
sudo apt-get install asciidoc xmlto --no-install-recommends
{% endhighlight %}

BTRFS_BUILD VERSION should be set before install dependencies.

If not, <a href="https://git.kernel.org/cgit/linux/kernel/git/kdave/btrfs-progs.git/tree/version.h.in"> set correct version.</a>

{% highlight bash %}
$ #first check if your version is Btrfs v3.18.2
$ 
$ fgrep BTRFS_BUILD_VERSION btrfs-progs2/version.h
$ fgrep BTRFS_BUILD_VERSION version.h
{% endhighlight %}

Install dependencies for BTRFS driver.

{% highlight bash %}
$ git clone https://kernel.googlesource.com/pub/scm/linux/kernel/git/mason/btrfs-progs
$ cd btrfs-progs/
$ ./autogen.sh
$ ./configure
$ make
$ sudo make install
{% endhighlight %}

## Build Docker

{% highlight bash %}
$ git clone https://git@github.com/docker/docker
$ cd docker/
$ #Checkout on the last stable version
$ git branch release-v1.6.2
$ git checkout release-v1.6.2
$ git checkout v1.6.2
$ AUTO_GOPATH=1 ./hack/make.sh dynbinary
{% endhighlight %}

## Installation

{% highlight bash %}
{% endhighlight %}

{% highlight bash %}
$ sudo install -m 755 -o root -g root docker-1.6.2 /usr/bin/docker-1.6.2
$ sudo install -m 755 -o root -g root dockerinit-1.6.2 /usr/bin/dockerinit-1.6.2
$ cd /usr/bin/
$ sudo ln -s docker-1.6.2 docker
$ sudo ln -s dockerinit-1.6.2 dockerinit
{% endhighlight %}

## Check the Docker status and check syncronization with Udev

{% highlight bash %}
$ sudo service docker start
$ docker info | grep -i udev
Udev Sync Supported: true
WARNING: No swap limit support
$ docker version
Client version: 1.6.2
Client API version: 1.18
Go version (client): go1.2.1
Git commit (client): 7c8fca2
OS/Arch (client): linux/amd64
Server version: 1.6.2
Server API version: 1.18
Go version (server): go1.2.1
Git commit (server): 7c8fca2
OS/Arch (server): linux/amd64
{% endhighlight %}


Now, your Docker is running in accordance with Udev, should not have problems with devicemapper.

###See Ya




