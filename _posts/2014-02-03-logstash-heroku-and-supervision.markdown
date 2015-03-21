---
layout: post
category: programming
title:  "Logstash, Heroku, and supervision"
date:   2014-02-03
listed: false
permalink: /2014/02/03/logstash-heroku-and-supervision.html
published: true
---

*This post originally appeared on my old blog. Copied over a slightly revised on 2014-07-12 (1405196000).*

For the last couple weeks in my spare time I've been trying to setup a logstash / elasticsearch / kibana box to manage our logs. Most of the work went pretty quickly, but then I've been stuck trying to get logstash to run under supervision. I finally made the breakthrough that allowed it to work, though, so I wanted to share it since it was hard to find anything online to help me.

One of the things I want logstash to do is pull logs from Heroku. In order for this to work, I need to install the Heroku toolbelt on the server and login. Then logstash will pick up the Heroku credentials and use them to grab the logs from specified apps in the logstash config.

Initially I seemed to have things working. I'd authenticate Heroku, start logstash, and see it pull down logs into elasticsearch. Great! Only logstash has bugs and doesn't seem to stay up for more than a couple hours, so I needed to run logstash under supervision. I tried both runit and daemontools, and with both of them I found that I could write a run script that worked fine if I ran it directly from the shell but would fail after about 20 seconds when used by the tools.

My problem was confounded because logstash has a `--log` option that I was using and those logs didn't indicate any errors. It was only after digging so deep into daemontools as to rule out anything else, getting on the #logstash IRC channel, and talking with others that I learned the importance of redirecting stderr to stdout when running a process under supervision and putting stdout somewhere I could read it. That quickly exposed my problem: Heroku was failing to authenticate.

Unfortunately, it's not all that clear how Heroku saves credentials, 
 but after some searching I found that it stores your username and a hash
  of your password in `HOME/.netrc`. And, since the run script has to setup whatever environment variables it needs, I needed to set `HOME` in my run script. One iteration later everything was working.

So if you are having trouble getting logstash (or probably any process, Java or otherwise) to run under supervision, make sure you are grabbing stderr and writing it somewhere you can see it.
