---
layout: post
category: programming
title:  "DbCharmer: Getting Started"
date:   2014-03-28
listed: false
permalink: /2014/03/28/dbcharmer-getting-started.html
published: true
---

*This post originally appeared on my old blog. Copied over a slightly revised on 2014-07-10 (1405034000).*

I recently integrated [DbCharmer](http://dbcharmer.net/) into our Rails application at [AdStage](http://adstage.io/). There's not a whole lot written about how to set it up, especially if you want to use it with something like [Sidekiq](http://sidekiq.org/), so here's a quick guide on how I did our setup using Ruby 2.0 and Rails 3.
>
First, you need to put DbCharmer in your Gemfile:

```ruby
gem 'db-charmer', '~>1.9.0'
```

Next you'll want to initialize DbCharmer to know about your shards or read slaves. Because you should not put configuration in code, we first get the slave database connection information in `config/initializers/db_charmer.rb`:

```ruby
SLAVE_DB_CONNECTIONS = (ENV['SLAVE_DATABASE_URLS'] || '').split(',').map do |db_url|
  ENV[db_url] && Class.new(ActiveRecord::Base) { establish_connection(ENV[db_url]) }.connection
end.compact
```

This gives us a constant we can use to pass to DbCharmer later. Yes, I agree, this is a little convoluted, but DbCharmer expects connection objects, not connection strings, so you need to do something like this.

Next, to turn DbCharmer on you call the `db_magic` function from your `ActiveRecord` subclass definition. Or, if you just want to have access to it everywhere, because you almost certainly do, you can extend `ActiveRecord::Base` to inject `DbCharmer` everywhere:

```ruby
require 'db_charmer'

class ActiveRecord::Base
  extend AdStage::Logger
  include AdStage::Logger
  
  db_magic :slaves => (SLAVE_DB_CONNECTIONS.empty? ? [connection] : SLAVE_DB_CONNECTIONS), :force_slave_reads => false
  log(:warn, "no slave databases connected") if SLAVE_DB_CONNECTIONS.empty?
end
```

A couple of things to note. First, there's some logic to handle the case that `SLAVE_DB_CONNECTIONS` is empty since that will happen in development and test where the environment variable `SLAVE_DATABASE_URLS` is likely not set. Since in that case there is no slave, it just reuses the master connection as the slave connection. This has a side benefit of making it super easy to turn read slave use on and off. However, because you don't want to fail to notice if read slaves are not set up correctly in production, a warning is raised you can check the logs for when `SLAVE_DB_CONNECTIONS` is empty.

We also set read slaves off by default. To me this makes a lot of sense because now existing code will continue to work as before without changes: everything you have already written will still go against master every time. But, now you can easily choose to use read slaves whenever you want via DbCharmer's convenient block form:

```ruby
DbCharmer.force_slave_reads do
  # all read operations againt ActiveRecord objects will now
  #   go against the read slaves. this extends to calls on
  #   ActiveRecord objects in nested function calls.
end
```

This is extremely useful for processes like reports that don't ever write to the database (or at least not until the end) and won't suffer much if the read slaves fall a few commits behind the master. Since complex reports can put a lot of strain on your database, offloading the work to slaves increases the capacity you have available for handling interactive user requests that often create or modify database rows.

One last thing: we had some trouble with DbCharmer and Sidekiq because Sidekiq is multithreaded and DbCharmer is only partially thread safe. We ran into race conditions using the existing 1.9.0 release, but made a simple patch and it works well for us in production. As of the time of writing, you can get thread-safer DbCharmer by applying my [pull request](https://github.com/kovyrin/db-charmer/pull/94/files) against DbCharmer or just grab [the AdStage fork](https://github.com/AdStage/db-charmer).
