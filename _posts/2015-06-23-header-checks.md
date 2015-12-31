---
layout: post
title: "Header Checks: Everyone But Us"
description: "A specific postfix filtering use case which caused minor confusion."
category: tech
tags: [tech, computers, postfix, header_check]
---
{% include JB/setup %}

Here's the scenario: I have a fancy relay, which I want to use for externally-bound email. Unfortunately, I have a single smtp server that handles both internal and external mail. This general problem is well covered - [Marcelog](http://marcelog.github.io/articles/configure_postfix_forward_email_regex_subject_transport_relay.html)'s guide was particularly helpful - I decided to use a header check.

In `/etc/postfix/main.cf`, I let postfix know that I'll be using a header check. 

```
header_checks = pcre:/etc/postfix/header_checks
```

I don't have an exhaustive list of external domains to filter against, so I look for anything *not* going to a corporate account. Here's what I stick in `/etc/postfix/header_checks`

```
!/^[tT]o:.*@moopinc\.com/ FILTER relay:smtp-relay.fancyshmancyrelay.com
```

Now, I apply this change (`postmap hash:/etc/postfix/header_checks`) and reload postfix (`postfix reload`). Then I shot off emails to my corporate account and personal accounts.

```
$ mailx -s 'Testing' jmhbrown@moopinc.com
$ mailx -s 'Testing' jmhbrown@zerpmail.com
```

Only, both those messages go through the fancy relay. I changed the FILTER in my header checks to a WARN, and try again

```
# Use WARN to help debug
!/^[tT]o:.*@moopinc\.com/ WARN EXTERNAL Address
/^[tT]o:.*@moopinc\.com/ WARN MOOPINC Address
```


Here's what I find in `/var/log/maillog`:

```
2015-06-23T15:51:47.635874-04:00 test-smtp-server postfix/postfix-script[26765]: refreshing the Postfix mail system
2015-06-23T15:51:47.644627-04:00 test-smtp-server postfix/master[28112]: reload -- version 2.6.6, configuration /etc/postfix
2015-06-23T15:51:56.521383-04:00 test-smtp-server postfix/pickup[26770]: 7F39E20721: uid=0 from=<root>
2015-06-23T15:51:56.527978-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header Received: by test-smtp-server (Postfix, from userid 0)??id 7F33439539; Tue, 23 Jun 2015 15:51:56 -0400 (EDT) from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: EXTERNAL Address
2015-06-23T15:51:56.528001-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header Date: Tue, 23 Jun 2015 15:51:56 -0400 from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: EXTERNAL Address
2015-06-23T15:51:56.528500-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header To: jmhbrown@moopinc.com from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: MOOPINC Address
2015-06-23T15:51:56.528520-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header Subject: Testing from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: EXTERNAL Address
2015-06-23T15:51:56.528526-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header User-Agent: Heirloom mailx 12.4 7/29/08 from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: EXTERNAL Address
2015-06-23T15:51:56.528531-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header MIME-Version: 1.0 from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: EXTERNAL Address
2015-06-23T15:51:56.528537-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header Content-Type: text/plain; charset=us-ascii from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: EXTERNAL Address
2015-06-23T15:51:56.528542-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: warning: header Content-Transfer-Encoding: 7bit from local; from=<root@test-smtp-server> to=<jmhbrown@moopinc.com>: EXTERNAL Address
2015-06-23T15:51:56.528548-04:00 test-smtp-server postfix/cleanup[26774]: 7F39E20721: message-id=<20150623195156.7FSKJSDFKS@test-smtp-server>
```

Ah, that makes sense. The header spans multiple lines and it seems that a match on any line in the header is enough to trigger the filter. If we switch back to our original header_checks, it's obvious that the regex is a little too eager. 

```
$ postmap -q 'Subject: Testing' pcre:/etc/postfix/header_checks
FILTER relay:smtp-relay.fancyshmancyrelay.com
```

So, I needed to look for header lines that start with 'To:' and don't include 'moopinc.com'. There are a millions ways to do that, here's what I settled on:

```
if /[tT]o:/
!/.*@(|[a-z]+\.)moopinc\./ FILTER relay:smtp-relay.fancyshmancyrelay.com
endif
```
