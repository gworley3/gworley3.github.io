---
layout: post
title:  Tracking ad conversions using Cassandra
date:   2014-11-24
published: true
---

Last week I gave a talk at the [San Francisco Cassandra Meetup](http://www.meetup.com/CassandraSF/) about how we're using Cassandra at [AdStage](http://adstage.io) to track ad conversions. Lina, the meetup's organizer, was kind enough to record the presentation, so it's now available on YouTube.

<iframe width="560" height="315" src="//www.youtube.com/embed/VDM8PPxPEYo" frameborder="0" allowfullscreen></iframe>

Just to add a couple of interesting details here, first, here are the schemas for the tables I discussed in the talk so you can get a better idea of what they look like:


```
CREATE TABLE goal (
  tracker_id text,
  goal_id text,
  field ascii,
  operation ascii,
  value text,
  PRIMARY KEY ((tracker_id), goal_id)
);

CREATE TABLE click (
  click_id text,
  facet_ids list<text>,
  PRIMARY KEY ((click_id))
);

CREATE TABLE goal_member (
  goal_id text,
  facet_id text,
  member_id text,
  PRIMARY KEY ((goal_id, facet_id, member_id))
);

CREATE TABLE goal_conversion (
  goal_id text,
  facet_id text,
  ts timestamp,
  "count" counter,
  PRIMARY KEY ((goal_id, facet_id), ts)
);
```

And because it got cutoff in the video, here's the full diagram of how the tables are used from the slides.

![conversion tracker pixel processing diagram](/assets/conversion-tracker-pixel-processing-diagram.png)

And a quick shout-out to my coworker [Eric](https://twitter.com/ciretsof) because, if you look at the schemas above and are wondering "why is the primary key for a table like click written as `((click_id))` now instead of `(click_id)`?", it's because one day I was using Cassandra, failed to notice that `PRIMARY KEY (partition_key, cluster_key)` means 'partition by `primary_key`, cluster by `cluster_key`' rather than 'partition by `primary_key` and `cluster_key`' and saw some unexpected behavior. With Eric's help, though, we proposed a change that was merged into Cassandra 2.0.x so that the partition keys are always surrounded by parentheses even when the partition key is not compound to make the distinction clear.
