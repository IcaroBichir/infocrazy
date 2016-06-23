---
layout: post
title: Use Postfix With AWS SES or Gmail
modified:
categories: 
excerpt: "Configure postfix to send mail with SES or Gmail"
tags: [aws, ses, postfix, mail, gmail]
comments: true
image:
  feature: sample-image-4.png
  credit: BlueBoy
  creditLink: http://www.blueboyimaging.com/direct-mail/
date: 2016-06-22T22:49:59-03:00
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


> Works on Centos and Ubuntu

> Execute all steps on root account

### Before you start: 

> Generate your SMTP username and password ( <a href="http://docs.aws.amazon.com/ses/latest/DeveloperGuide/smtp-credentials.html" target="_blank">AWS SES</a> ; <a href="https://security.google.com/settings/security/apppasswords" target="_blank">Gmail, will need App password</a>

> <a href="http://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses.html" target="_blank">Verify</a> the domain and email addresses

> Check if your *certificate* exists 

- Centos: */etc/pki/tls/certs/ca-bundle.crt*

- Ubuntu: */etc/ssl/certs/ca-certificates.crt*

> This tutorial was made for *us-east* region, just change the <a href="http://docs.aws.amazon.com/ses/latest/DeveloperGuide/smtp-credentials.html" target="_blank">endpoint for other regions</a>

### Install the required packages

#### Centos

{% highlight bash %}
$ yum install postfix mailx cyrus-sasl cyrus-sasl-plain cyrus-sasl-lib cyrus-imapd cyrus-imapd-utils
{% endhighlight %}

#### Ubuntu

{% highlight bash %}
$ apt-get install postfix mailutils libsasl2-2 ca-certificates libsasl2-modules
{% endhighlight %}

### Edit the postfix configuration with SMTP settings and insert this lines on the botton of the */etc/postfix/main.cf* file (On Ubuntu, just change the *.crt* directory)

#### Gmail

{% highlight bash %}
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = secure
smtp_tls_mandatory_protocols = TLSv1
smtp_tls_mandatory_ciphers = high
smtp_tls_secure_cert_match = nexthop
smtp_tls_CAfile = /etc/pki/tls/certs/ca-bundle.crt
relayhost = smtp.gmail.com:587
{% endhighlight %}

#### AWS SES

{% highlight bash %}
relayhost = [email-smtp.us-east-1.amazonaws.com]:25
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_use_tls = yes
smtp_tls_security_level = encrypt
smtp_tls_note_starttls_offer = yes
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
{% endhighlight %}

### Create the *sasl_passwd* file with SMTP username and password

#### Gmail
{% highlight bash %}
$ touch /etc/postfix/sasl_passwd
$ cat << EOF >/etc/postfix/sasl_passwd
[smtp.gmail.com]:587  USERNAME@gmail.com:PASSWORD
EOF
{% endhighlight %}

#### AWS SES 
{% highlight bash %}
$ touch /etc/postfix/sasl_passwd
$ cat << EOF >/etc/postfix/sasl_passwd
[email-smtp.us-east-1.amazonaws.com]:25 SMTP_USERNAME:SMTP_PASSWORD
[ses-smtp-prod-335357831.us-east-1.elb.amazonaws.com]:25 SMTP_USERNAME:SMTP_PASSWORD
EOF
{% endhighlight %}

### Change the permition and generate the password file for postfix

{% highlight bash %}
sudo chmod 400 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
{% endhighlight %}

### Restart postfix
{% highlight bash %}
service postfix restart
{% endhighlight %}

### Check if email are sent

- You can check your log for more information ~~( /var/log/maillog )~~

#### Gmail
{% highlight bash %}
echo "Test mail from postfix" | mail -s "Test Postfix" you@example.com
{% endhighlight %}

#### AWS SES
{% highlight bash %}
sendmail -f from@example.com to@example.com
From: from@example.com
Subject: Test
This email was sent through Amazon SES!
.
{% endhighlight %}

### Cool, It's done ~~(I hope so)~~

If you have some question or update about this procedure, please contact me.

