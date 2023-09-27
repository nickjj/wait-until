# wait-until ![CI](https://github.com/nickjj/wait-until/workflows/CI/badge.svg?branch=master)

A zero dependency Bash script that waits until a command of your choosing has
run successfully.

- [Demo Video](#demo-video)
- [Use Cases](#use-cases)
- [Installation](#installation)
- [Usage Examples](#usage-examples)
- [Configuration Options](#configuration-options)
- [About the Author](#about-the-author)

## Demo Video

This video covers why you might want to use this script and how to use it. It's
basically the documention in video form.

[![Demo
video](https://nickjanetakis.com/assets/blog/cards/wait-until-your-dockerized-database-is-ready-before-continuing-6181d654c8c7a45dfffe0384afacb58e71db84fe75bac9eceb53a90faa122159.jpg)](https://www.youtube.com/watch?v=jqqIQoSpxxA)

## Use Cases

I mainly use this in my CI pipelines to wait until a Dockerized database is
ready to accept connections to a specific database as a specific user.

For example, you might have something like this in your CI set up:

```sh
docker-compose up -d

make db-reset
make lint
make test
```

The make commands don't really matter, but the idea is you can't really reset
your database or run your tests that depend on your database being set up
before your database is available.

When you first start the official PostgreSQL or MySQL Docker containers, it
will take about 5 to 10 seconds for your initial database user / database to
be created based on environment variables you pass into the container.

This isn't a problem in development because you'd probably run everything in
the foreground and manually wait until your DB is ready before interacting
with your project. But in a CI environment, it's a problem because it will
blow up without waiting.

Technically you can use this script to wait for any command to run successfully.
There is nothing about it that's limited to databases or CI environments. There
are [usage examples](#usage-examples) included in the README.

### Why not sleep X seconds?

You could, but that's kind of brittle because depending on where you run the
command, it might take a different amount of time to be ready since the spin up
time of a command is based on the hardware specs of the machine.

If you put a long sleep like 30 seconds to be safe then you're waiting
potentially tens of extra seconds on every CI run.

### Can't you do `docker-compose up -d && make db-reset`?

Technically yes, but you're going to be at the mercy of race conditions. That
isn't going to ensure that your database is "really" ready before control is
passed over to the 2nd command in the chain.

I ran that 10 times manually and it failed 9 out of 10 times. That's not really
suitable to have running in CI.

### What about the `wait-for-it` script?

The [wait-for-it](https://github.com/vishnubob/wait-for-it) script is popular,
but in my opinion it's not suitable for the CI use case and it solves a
different use case.

For starters, it waits for a TCP port to be available. Technically that port
can be available shortly after your database service is up, but your DB is not
really ready for connections because the DB user and database itself hasn't
been created yet.

That could potentially cause your CI run to fail.

Another issue with that script is that it expects you to access a TCP port,
which means you'll need to run it inside of your app's container or publish
your database's port back to your Docker host if you wanted to use it for CI.

If you're using that script to make `depends_on` more robust, then yes you'll
have that in your app's Docker image but personally I don't do that. IMO that
problem should be solved at the web framework level. Once the DB itself is
created with the proper user credentials your app's code should be robust
enough to retry your database connection until it works or gives up.

As for publishing the port back to the host so you can run the script from
within your CI server, that comes with 2 problems:

1. Now you're responsible for having to install the `psql` or `mysql` CLI directly within your CI environment.
2. You'll need to publish your database's port in your `docker-compose.yml` file to access your DB's TCP port.

Personally I don't like either of those things. The second one means allowing
the public internet to attempt to login to your database so having that in my
real compose file by default isn't happening and I didn't want to have to rig
up some `sed` in-line replacement script to do the port publish specifically
for CI (basically editing the file only for CI).

And that's why I decided to make this script.

## Installation

Typically you would install this script into a base Docker image that you use
as your CI environment, but we'll cover a few installation methods here.

The snippets below are set to use the latest release. If you're living on the
edge you can always replace the tag name with `master`, but I don't recommend
that since it's not guaranteed to be stable and may change in the future.

### Installing it directly on your system

This allows you to run `wait-until` from any directory. If this is on your
personal dev box or something like that, feel free to adjust this to put it
into a local `bin/` directory that's on your path for your user.

```sh
sudo curl \
  -L https://raw.githubusercontent.com/nickjj/wait-until/v0.3.0/wait-until \
  -o /usr/local/bin/wait-until && sudo chmod +x /usr/local/bin/wait-until
```

### Installing it remotely within a Docker image (useful for CI images)

You would add these instructions somewhere near the bottom of your `Dockefile`.

```Dockerfile
ADD https://raw.githubusercontent.com/nickjj/wait-until/v0.3.0/wait-until /usr/local/bin
RUN chmod +x /usr/local/bin/wait-until
```

This is one of those times where [using `ADD` instead of
`COPY`](https://nickjanetakis.com/blog/docker-tip-2-the-difference-between-copy-and-add-in-a-dockerile)
comes in handy.

### Installing it locally within a Docker image (useful for CI images)

If you're going for a super optimized base CI image and you already have your
own scripts being copied to `/usr/local/bin` and you want to piggy back off
your `COPY usr/local/bin /usr/local/bin` instruction you may want to just copy
/ paste this script directly into your image.

This 1 liner will download the latest release into the current directory of
your system. Then you can put it anywhere you want.

```sh
curl \
  -L https://raw.githubusercontent.com/nickjj/wait-until/v0.3.0/wait-until \
  -o wait-until && chmod +x wait-until
```

## Usage Examples

You can use this for any command but here's a couple of database related examples.

### A quick note about .env files and database passwords

In all examples, we're sourcing an `.env` file because typically that's where
you would define your database environment variables that the official Docker
images expect to be set.

Your `.env` file is usually ignored from version control which is a good idea.
What I do is commit a safe `.env.example` file to version control and then `mv
.env.example .env` as part of my CI pipeline. This file would have development
credentials that work and are safe to commit as defaults.

But if you're connecting to a remote database with sensitive credentials, then
you may want to skip that source step and read those environment variables
directly in from your CI provider's secure environment variable settings.

### Waiting for PostgreSQL

This expects that you've named your Docker Compose PostgreSQL service
`postgres` if you plan to copy / paste what's below.

```sh
docker-compose up -d

source .env
wait-until "docker-compose exec -T -e PGPASSWORD=${POSTGRES_PASSWORD} postgres psql -U ${POSTGRES_USER} ${POSTGRES_USER} -c 'select 1'"

# TODO: Run your DB reset commands, test suite, etc.
```

### Waiting for MySQL

This expects that you've named your Docker Compose MySQL service `mysql` if you
plan to copy / paste what's below.

```sh
docker-compose up -d

source .env
wait-until "docker-compose exec -T -e MYSQL_PWD=${MYSQL_ROOT_PASSWORD} mysql mysql -D ${MYSQL_DATABASE} -e 'select 1'"

# TODO: Run your DB reset commands, test suite, etc.
```

#### What's with the `-T` flag?

By default `docker-compose exec` configures a TTY which is fine and dandy in
development since you're running a terminal emulator. This has nice advantages
like being able to have an interactive prompt and see colors.

Within most (maybe all?) CI environments you'll get a `the input device is not
a TTY` error without having `-T` set when using this script. That flag disables
pseudo-tty allocation.

#### The failed connection output is noisy, make it stop!

You can always adjust your command to redirect everything to `/dev/null`. For
example, at the end of your command (outside of the quotes) you can add `>
/dev/null 2>&1` to eliminate all output (both STDOUT and STDERR).

Personally I like seeing all of the output in CI. It's especially handy for
being able to debug the command you're waiting on. For example I initially
redirected everything to `/dev/null` by default but it made debugging that `-T`
issue a pain in the butt.

## Configuration Options

You can add an optional second argument to customize the timeout in seconds. By
default it's set to 60 seconds.

For example you can run `wait-until "grep" 3` to try it out. The `grep` command
will fail to run since it requires at least 1 argument. As configured
`wait-until` will try for 3 seconds until it gives up.

## About the Author

I'm a self taught developer and have been freelancing for the last ~20 years.
You can read about everything I've learned along the way on my site at
[https://nickjanetakis.com](https://nickjanetakis.com/). There's hundreds of
[blog posts](https://nickjanetakis.com/blog/) and a couple of [video
courses](https://nickjanetakis.com/courses/) on web development and deployment
topics. I also have a [podcast](https://runninginproduction.com) where I talk
to folks about running web apps in production.
