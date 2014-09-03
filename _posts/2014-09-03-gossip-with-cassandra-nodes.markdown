---
layout: post
title:  Gossip with Cassandra Nodes
date:   2014-09-03
published: true
---

The trouble with gossip is that sometimes false information gets spread. Try as you might to correct it, the false information persists and only goes away after everyone has had enough time to forget about it.

I'm, of course, talking about the gossip protocol in a Cassandra cluster. Sometimes when a node dies or leaves, one or more nodes will fail to learn that the node has left and so will persist in believing that the node is part of the cluster in gossip, even if outwardly it appears that the node has disconnected (it doesn't show up in any rings when you use `nodetool status`).

You can check the state of gossip by running `nodetool gossipinfo`. This will show you what that node knows about via the gossip protocol. You'll see a number of entries like the following one.

```
/10.237.162.179
  RPC_ADDRESS:0.0.0.0
  RELEASE_VERSION:2.0.7.31
  HOST_ID:b8e09660-432b-401c-94e9-41a0a4fe198c
  SEVERITY:0.0
  RACK:1a
  SCHEMA:6886fc5b-c6af-3b8b-8327-9b659965fc07
  LOAD:1.008436042902E12
  X_11_PADDING:{"active":"true"}
  DC:us-east
  NET_VERSION:7
  STATUS:NORMAL,-1008002558822670144
```

When a node leaves the cluster its status in gossip should be changed to `LEFT`, but if that doesn't happen the node may be removed from the ring but still be believed to be present in gossip and showing `NORMAL` status.

This is a rather nasty problem to fix because it's not supposed to happen, so there's no nodetool command that will resolve it. Instead, you're going to have to get under the covers and spread some gossip yourself (or just wait for it to naturally drop out of gossip after about 3 days, which you probably don't have time for).

To do this we'll need to use JMX on the running Cassandra processes on the nodes.

To begin with, make sure that JMX is turned on in your `cassandra-env.sh`. You should find some lines in there like this that set the JVM options for JMX. If you don't, add them and restart the node (you may have to cycle the node a few times because gossip is messed up and it may not come up right the first time).

```sh
JVM_OPTS="$JVM_OPTS -Djava.net.preferIPv4Stack=true"
JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.port=7199"
JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.ssl=false"
JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.authenticate=false"
```

Of course you can turn on SSL and provide a password file for authentication, which you should probably do if you expose the JMX port, but most folks don't seem to do this and instead leave the port closed to the outside world. So instead we'll use a SOCKS proxy to get to JMX.

On your local machine, create a SOCKS proxy using ssh. You can use any port you like that isn't already in use. For example purposes I'm going to use 1234.

```sh
ssh -D 1234 cassandra.node.ip.or.dns.name
```

Then start up jconsole like this so that it will use the proxy and connect through to JMX on the Cassandra node.

```sh
jconsole -J-DsocksProxyHost=localhost -J-DsocksProxyPort=1234 service:jmx:rmi:///jndi/rmi://localhost:7199/jmxrmi
```

Jconsole should now launch and connect. The UI kind of sucks, so don't be surprised if it pops up stupid error messages that you should just click through or if it takes a long time to respond to requests without indicating that it's doing anything. Eventually you should see something like this:

![jconsole initial screen after connecting to cassandra node](/assets/cassandra-jconsole-initial.png)

Now click over to the MBeans tab and on the left tree navigate down through `org.apache.cassandra.net` to `Gossiper` to `Operations` and finally to `unsafeAssassinateEndpoint`.

![jconsole mbeans screen with unsafeAssassinateEndpoint selected](/assets/cassandra-jconsole-mbeans.png)

Now just replace `String` in the box with the IP address of the offending node in gossip and press the `unsafeAssassinateEndpoint` button. You'll probably get no response right away unless you did something wrong, so just wait and after 10 or 15 seconds you should get a popup saying the message was handled.

Now if you run `nodetool gossip` you should see that status of the bad node has changed to `LEFT`. Troublesome gossip resolved.

I had to piece this together from reading about [unsafeAssassinateEndpoint](http://tumblr.doki-pen.org/post/22654515359/assassinating-cassandra-nodes), addressing some [JMX gotchas](http://wiki.apache.org/cassandra/JmxGotchas), and figuring out how to [proxy into JMX](http://www.jethrocarr.com/2013/11/30/jconsole-to-remote-servers-easily/). I probably would have ended up on IRC begging for help if not for these useful links.
