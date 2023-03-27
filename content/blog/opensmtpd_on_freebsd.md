---
title: "OpenSMTPD on FreeBSD"
date: 2023-03-27
description: "How to forward local emails to Google Mail"
slug: "opensmtpd-on-freebsd"
tags: [ "freebsd", "opensmtpd" ]
categories: [ "FreeBSD" ]
---

The goal of this blog post is to show how to use OpenSMTPD on FreeBSD in order to deliver local mail to my Google Mail account. The blog post combines knowledge from the manual pages `smtpd(8)`, `smtpd.conf(5)` with an excellent wiki article listed in Resources section.

# Google Mail App Password
Any Google account that has 2-factor authentication enabled prohibits the so-called "less secure" apps from using the standard Google password. In order to overcome this problem, navigate to the Security page in Google Account and then move to Two-factor Authentication section. Generate a new special App Password for the service here.

# OpenSMTPD Configuration
Install the package by running `# pkg install opensmtpd` as root. The next step is to create a new file with password that we have created in previous step. Let's follow the steps from the manual page:

```
# touch /usr/local/etc/mail/secrets
# chmod 640 /usr/local/etc/mail/secrets
# chown root:_smtpd /usr/local/etc/mail/secrets
```

Put the following line into the newly created file at `/usr/local/etc/mail/secrets`. Replace the placeholders with their real values.

```
gmail <user>@gmail.com:<app_password>
```

Then create `aliases` file, so the daemon knows how to map our local root user. I decided to copy the original file from `/etc/mail/aliases` to `/usr/local/etc/mail/aliases`. Inside the copied file replace the following line with the real email address:

```
# root: me@my.domain
root: <user>@gmail.com
```

The final step is to edit the configuration file of the service itself: `/usr/local/etc/mail/smtpd.conf`.

```
#       $OpenBSD: smtpd.conf,v 1.10 2018/05/24 11:40:17 gilles Exp $

# This is the smtpd server system-wide configuration file.
# See smtpd.conf(5) for more information.

table aliases file:/usr/local/etc/mail/aliases
table secrets file:/usr/local/etc/mail/secrets

# To accept external mail, replace with: listen on all
#
listen on localhost

action "local" mbox alias <aliases>
action "relay" relay host smtps://gmail@smtp.gmail.com:465 auth <secrets>

# Uncomment the following to accept external mail for domain "example.org"
#
# match from any for domain "example.org" action "local"
match for local action "local"
match from local for any action "relay"
```

I prefer to leave most of the file intact as it makes future upgrades easier. The `table` section refers to our previously created files: `aliases` and `secrets`. Since we used label `gmail` in the `aliases` file, we have to use it here in the `relay` section. The local delivery goes to `mbox` file, but only for the mail that is not being forwarded to the SMTP server. In our case, it's every other user than `root`.

# System Configuration
Now we can finally disable the built-in `sendmail` service completely and enable the OpenSMTPD service instead. This can be done by editing `/etc/rc.conf` file accordingly:

```
smtpd_enable="YES"
sendmail_enable="NO"
sendmail_submit_enable="NO"
sendmail_outbound_enable="NO"
sendmail_msp_queue_enable="NO"
```

Stop the `sendmail` service first with `# service stop sendmail` and then start the OpenSMTPD service with `# service start smtpd`. You can also check the logs in the usual place: `$ tail -f /var/log/maillog`. Try to send an email to Charlie Root: `echo "It works!" | mail -s "Test from FreeBSD" root`. If you did everything correctly, you should receive an email in your inbox immediately. Well done!

# Resources
I used following blog posts and sites to learn about FreeBSD mail delivery system, OpenSMTPD and on how to use it in conjunction with Google Mail.

* [FreeBSD Handbook - Electronic Mail](https://docs.freebsd.org/en/books/handbook/mail/) - how to disable `sendmail`
* [OpenSMTPD manual pages](https://www.opensmtpd.org/manual.html) - `smtpd(8), smtpd.conf(5), smtpctl(8)` and others
* [OpenSMTPD forward to Google](https://xw.is/wiki/OpenSMTPD_forward_to_Google) - Wiki page for OpenBSD installation, main source of my inspiration for this blog post
* [Google - Sign in with App Passwords](https://support.google.com/accounts/answer/185833?hl=en) - badly maintained help page from Google
