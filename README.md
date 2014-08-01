# ActiveRecord::Bogacs

ActiveRecord pooling "hacks" ... in a relaxed 'spa' fashion!

![Bogacs][0]

Bogács is a village in Borsod-Abaúj-Zemplén county, Hungary.

**WiP: don't even thing about putting this on production (TEST IT OUT FIRST)!**

## Install

add this line to your application's *Gemfile*:

    gem 'activerecord-bogacs', :require => 'active_record/bogacs'

... or install it yourself as:

    $ gem install activerecord-bogacs

## Setup

Bogacs' pools rely on a small monkey-patch that allows to change the AR pool.
The desired pool class needs to be set before `establish_connection` happens,
thus an initializer won't work, you might consider setting the pool class at
the bottom of your *application.rb* e.g. :

```ruby
Bundler.require(*Rails.groups)

module MyApp
  class Application < Rails::Application
    # config.middleware.delete 'ActiveRecord::QueryCache'
  end
end

# sample configuration using the "false" pool
if Rails.env.production?
  pool_class = ActiveRecord::Bogacs::FalsePool
  ActiveRecord::ConnectionAdapters::ConnectionHandler.connection_pool_class = pool_class
end
```

pools are expected to work with older ActiveRecord versions, if not let us know.

### Default Pool

Meant as a back-port for users stuck with old Rails versions (< 4.0) on production,
facing potential (pool related) concurrency bugs e.g. with high-loads under JRuby.

Based on pool code from 4.x (which works much better than any previous version),
with a few minor tunings and extensions such as `pool_initial: 0.5` which allows
to specify how many connections to prefill when the pool is created.

### False Pool

The false pool won't do any actual pooling, it is assumes that an underlying pool
is configured. But it does maintain a cache of AR connections mapped to threads.
Ignores all pool related configuration `pool: 42`, `checkout_timeout: 2.5` etc.

**NOTE:** be sure to configure an underlying pool e.g. with Trinidad (using the
default Tomcat JDBC pool) :

```yaml
---
  http: # true
    port: 3000
    # ...
  extensions:
    mysql_dbpool:
      url: jdbc:mysql:///my_production
      username: root
      jndi: jdbc/BogacsDB
      initialSize: <%= ENV['POOL_INITIAL'] || 25 %> # connections created on start
      maxActive: <%= ENV['POOL_SIZE'] || 100 %> # default 100 (AR pool: size)
      maxIdle: <%= ENV['POOL_SIZE'] || 100 %> # max connections kept in the pool
      minIdle: <%= ENV['POOL_SIZE'] || 50 %>
      # idle connections are checked periodically (if enabled) and connections
      # that been idle for longer than minEvictableIdleTimeMillis will be released
      minEvictableIdleTimeMillis: <%= 3 * 60 * 1000 %> # default 60s
      # AR checkout_timeout: 5
      maxWait: <%= (( ENV['POOL_TIMEOUT'] || 5.0 ).to_f * 1000).to_i %> # default 30s
```

ActiveRecord-JDBC adapter allows you to lookup connection from JNDI using the
following configuration :

```
production:
  adapter: mysql2
  jndi: java:/comp/env/jdbc/BogacsDB
```

**NOTE:** using the `FalsePool` there's nothing to configure (in *database.yml*)
that is `no_pool: 5` or `checkout_timeout: 5` defaults!

### Shareable Pool

This pool allows for a database connection to be "shared" among threads, this is
very **dangerous** normally. You will need to understand the underlying driver's
connection whether it is thread-safe.

You'll need to manually declare blocks that run with a shared connection (**make
sure** only read operations happen within such blocks) similar to the built-in
`with_connection` e.g. :

```ruby
cache_fetch( [ 'user', user_id ] ) do
  ActiveRecord::Base.with_shared_connection { User.find(user_id) }
end
```

The pool will only share connections among such blocks and only if it runs out
of all connections (until than it will always prefer checking out a connection
just like a "regular" pool).

Only tested with ActiveRecord-JDBC-Adapter using the official Postgres' driver.

## Copyright

Copyright (c) 2014 [Karol Bucek](http://kares.org).
See LICENSE (http://en.wikipedia.org/wiki/MIT_License) for details.

[0]: http://res.cloudinary.com/kares/image/upload/c_scale,h_600,w_800/v1406451696/bogacs.jpg