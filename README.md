Heroku buildpack: pgbouncer
=========================

This is a [Heroku buildpack](http://devcenter.heroku.com/articles/buildpacks) that
allows one to run pgbouncer and stunnel in a dyno alongside application code.
It is meant to be used in conjunction with other buildpacks as part of a
[multi-buildpack](https://github.com/ddollar/heroku-buildpack-multi).

The primary use of this buildpack is to allow for transaction pooling of
PostgreSQL database connections among multiple workers in a dyno. For example,
10 unicorn workers would be able to share a single database connection, avoiding
connection limits and Out Of Memory errors on the Postgres server.

It uses [stunnel](http://stunnel.org/) and [pgbouncer](http://wiki.postgresql.org/wiki/PgBouncer).


FAQ
----
- Q: Why should I use transaction pooling?
- A: You have many workers per dyno that hold open idle Postgres connections and
and you want to reduce the number of unused connections. [This is a slightly more complete answer from stackoverflow](http://stackoverflow.com/questions/12189162/what-are-advantages-of-using-transaction-pooling-with-pgbouncer)

- Q: Why shouldn't I use transaction pooling?
- A: If you need to use named prepared statements, advisory locks, listen/notify, or other features that operate on a session level.
Please refer to PGBouncer's [feature matrix](http://wiki.postgresql.org/wiki/PgBouncer#Feature_matrix_for_pooling_modes) for all transaction pooling caveats.


Disable Prepared Statements
-----
With Rails 4.1, you can disable prepared statements by appending `?prepared_statements=false` to the database's URI.
Set the `PGBOUNCER_PREPARED_STATEMENTS` config var to `false` for the buildpack to do that for you.

For Rails 3.2 - 4.0 you will first need to apply a simplified backport of [this commit](https://github.com/rails/rails/commit/e54acf1308e2e4df047bf90798208e03e1370098). Just create a new ruby file in `config/initializers` with the following content:

    require "active_record/connection_adapters/postgresql_adapter"

    class ActiveRecord::ConnectionAdapters::PostgreSQLAdapter
      alias initialize_without_config_boolean_coercion initialize
      def initialize(connection, logger, connection_parameters, config)
        if config[:prepared_statements] == 'false'
          config = config.merge(prepared_statements: false)
        end
        initialize_without_config_boolean_coercion(connection, logger, connection_parameters, config)
      end
    end

and follow the instructions for Rails 4.1.


Usage
-----

Example usage:

    $ ls -a
    .buildpacks  Gemfile  Gemfile.lock  Procfile  config/  config.ru

    $ heroku config:add BUILDPACK_URL=https://github.com/ddollar/heroku-buildpack-multi.git

    $ cat .buildpacks
    https://github.com/gregburek/heroku-buildpack-pgbouncer.git#v0.2.1
    https://github.com/heroku/heroku-buildpack-ruby.git

    $ cat Procfile
    web:    bin/start-pgbouncer-stunnel bundle exec unicorn -p $PORT -c ./config/unicorn.rb -E $RACK_ENV
    worker: bundle exec rake worker

    $ git push heroku master
    ...
    -----> Fetching custom git buildpack... done
    -----> Multipack app detected
    =====> Downloading Buildpack: https://github.com/gregburek/heroku-buildpack-pgbouncer.git
    =====> Detected Framework: pgbouncer-stunnel
           Using pgbouncer version: 1.5.4
           Using stunnel version: 4.56
    -----> Fetching and vendoring pgbouncer into slug
    -----> Fetching and vendoring stunnel into slug
    -----> Moving the configuration generation script into app/.profile.d
    -----> Moving the start-pgbouncer-stunnel script into app/bin
    -----> pgbouncer/stunnel done
    =====> Downloading Buildpack: https://github.com/heroku/heroku-buildpack-ruby.git
    =====> Detected Framework: Ruby/Rack
    -----> Using Ruby version: ruby-1.9.3
    -----> Installing dependencies using Bundler version 1.3.2
    ...

The buildpack will install and configure pgbouncer and stunnel to connect to
`DATABASE_URL` over a SSL connection. Prepend `bin/start-pgbouncer-stunnel`
to any process in the Procfile to run pgbouncer and stunnel alongside that process.


Multiple Databases
----
It is possible to connect to multiple databases through pgbouncer by setting
`PGBOUNCER_URLS` to a list of config vars. Example:

    $ heroku config:add PGBOUNCER_URLS="DATABASE_URL HEROKU_POSTGRESQL_ROSE_URL"
    $ heroku run bash

    ~ $ env | grep 'HEROKU_POSTGRESQL_ROSE_UR\|DATABASE_URL'
    HEROKU_POSTGRESQL_ROSE_URL=postgres://u9dih9htu2t3ll:password@ec2-107-20-228-134.compute-1.amazonaws.com:5482/db6h3bkfuk5430
    DATABASE_URL=postgres://uf2782hv7b3uqe:password@ec2-50-19-210-113.compute-1.amazonaws.com:5622/deamhhcj6q0d31

    ~ $ bin/start-pgbench-stunnel env # filtered for brevity
    HEROKU_POSTGRESQL_ROSE_URL=postgres://u9dih9htu2t3ll:password@127.0.0.1:6000/db6h3bkfuk5430
    DATABASE_URL=postgres://uf2782hv7b3uqe:password@127.0.0.1:6000/deamhhcj6q0d31

Tweak settings
-----
Some settings are configurable through app config vars at runtime. Refer to the appropriate documentation for
[pgbouncer](http://pgbouncer.projects.pgfoundry.org/doc/config.html#_generic_settings)
and [stunnel](http://linux.die.net/man/8/stunnel) configurations to see what settings are right for you.

- `PGBOUNCER_POOL_MODE` Default is transaction
- `PGBOUNCER_DEFAULT_POOL_SIZE` Default is 1
- `PGBOUNCER_RESERVE_POOL_SIZE` Default is 1
- `PGBOUNCER_RESERVE_POOL_TIMEOUT` Default is 5.0 seconds
- `PGBOUNCER_URLS` Default is DATABASE_URL

For more info, see [CONTRIBUTING.md](CONTRIBUTING.md)
