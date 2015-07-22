---
layout: post
title: "Pull Request Validation Between Jenkins and Bitbucket"
modified:
categories: posts
excerpt: "Shell script to build every bitbucket pull request on Jenkins and comment the result"
tags: [Jenkins, Bitbucket, PR, Pull request, coverage, pylint, python, ruby]
comments: true
image:
  feature: sample-image-10.jpg
  credit:
  creditlink:
date: 2015-06-27T10:11:10-03:00
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

In the end of this tutorial, your Jenkins will execute every Bitbucket open pull request and post a coment with the commit number, link to console output, link to coverage report. 

------

Go to my <a href="https://github.com/IcaroBichir/jenkins_bitbucket_pullrequest" target="_blank"> Github</a> page and fork the master version of shell script.

> Your comments on bitbucker will be just like this:

![bitbucket_link]({{ site.url }}/images/jenkins-bitbucket/jenkins-004.png)


-------

### Configuration

Create a new job on your Jenkins and select the option: 

> " Build multi-configuration project "

#### **Settings Screen**

Inside GitBucket config, select **"This build is parameterized"** and create one parametrized value for each variable above:

* OWNER = bitbucket owner
* BUCKET_API_V1_URL = bitbucket.org/api/1.0/repositories
* BUCKET_API_V2_URL = api.bitbucket.org/2.0/repositories
* REPO = $repository
* USERNAME_BITBUCKET = username
* PASSWORD_BITBUCKET = p4ssw0rd
* USERNAME_JENKINS = username
* PASSWORD_JENKINS = p4ssw0rd
* JENKINS_URL = your_jenkins_url.com
* LANGUAGE = program language (python or ruby)

#### **"Source Code Management"**

Select **"Git"** and insert your bitbucket link with this pullrequest script.

![bitbucket_link]({{ site.url }}/images/jenkins-bitbucket/jenkins-001.png)

#### **"Build Triggers"**

Select **"Poll SCM"** and insert your crontab to Jenkins validade with bitbucet if there is a open PR. In my project, the Jenkins validate every 5 minutes.

{% highlight bash %}
H/5 * * * * 
{% endhighlight %}

#### **"Configuration Matrix"**

Create a **"User Defined Axis"** with the **"name: repository"** and on the values, inseert your Bitbucket repository name.

![bitbucket_link]({{ site.url }}/images/jenkins-bitbucket/jenkins-002.png)

 
#### **"Build"** 

Include this script on the **"Execute Shell"**

{% highlight bash %}
#!/bin/bash

./pr-response.sh
{% endhighlight %}

### Configure Code analisys 

#### **"Post Build Actions"**

> On Python

Activate the Cobertura Plugin **"Publish cobertura coverage Report"** and insert the location of the coverage report, on .xml format.

* **/coverage.xml

![bitbucket_link]({{ site.url }}/images/jenkins-bitbucket/jenkins-003.png)

> On Ruby

The ruby coverage will be generated on HTML format.

----------


# This script will bee function only if you are using python or ruby language and configure coverage report on  your tests. 
{: .notice}

---------

### VERY Important Note

> ci.sh is the file when I execute my environment configuration to run the tests.

{% highlight bash %}

## Your PYTHON .html coverage report MUST be on this path:
http://${JENKINS_URL}/job/${JOB_NAME}/repository=${REPOSITORY_NAME}/ws/${REPOSITORY_NAME}/htmlcov/index.html

## Your RUBY .html coverage report MUST be on this path:
http://${JENKINS_URL}/job/${JOB_NAME}/repository=${REPOSITORY_NAME}/ws/${REPOSITORY_NAME}/coverage/index.html

{% endhighlight %}

----------

####Thanks for your attention and be welcome to send me a message if you have any question or improvement to this post.

#See Ya
