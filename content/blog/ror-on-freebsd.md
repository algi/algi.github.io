---
title: "Ruby on Rails on FreeBSD"
date: 2025-04-25T22:35:26+02:00
description: "How I moved back to FreeBSD, installed Ruby on Rails 8 with Postgres."
slug: "ror-on-freebsd"
tags: [ "ror", "freebsd" ]
categories: [ "FreeBSD" ]
---

I decided to move back to FreeBSD after 2 years of running NixOS. One thing I appreciate on Linux is the ability to easily install Ruby on Rails framework. It's a framework of my choice for creating web applications, so naturally I was curious to see how Ruby will work on FreeBSD too. My hobby application is using Rails 8 and Postgres, so nothing too fancy. Let's deep dive into how it all went!

# Ruby, Rails and gems
Current version of Ruby on FreeBSD quarterly branch is 3.2, which means that all gems are compiled against this version. It is possible to switch to *latest* branch in order to get Ruby 3.4, which is a good news. In order to do that, create a new file called file: `/usr/local/etc/pkg/repos/FreeBSD.conf` with this content:

```
FreeBSD {
  url = "pkg+http://pkg.freebsd.org/\${ABI}/latest";
}
```

Before moving on, it's a good idea to create a new boot environment using `bectl create <snapshot>`. I haven't done that though as I had new installation on my laptop and therefore I had nothing to lose. The next step is to update the packages to the latest branch by `pkg upgrade -f`.

With that done, we can now install following packages:
```
ruby34
rubygems-gem
```

As I said before, Ruby gems are compiled against Ruby 3.2, so it will pull this dependency and will expose it as `/usr/local/bin/ruby`. In order to override that, we are going to change our `PATH` variable (line #2). By default Ruby gems installs into system directory, which will fail unless installed by *root*. I don't believe that it's a good idea, so I override the home location of gems (line #1) by setting different `GEM_HOME` variable. The last line is important for Cuprite gem to find correctly the browser I use for system testing.

All the following lines belongs to `.zshrc`:

```
export GEM_HOME=$(/usr/local/bin/ruby34 -e 'puts Gem.user_dir') # (1)
export PATH="$HOME/bin:$PATH:$GEM_HOME/bin"                     # (2)
export BROWSER_PATH="/usr/local/bin/ungoogled-chromium"         # (3)
```

Feel free to choose whatever browser you please, of course. With that, we have everything done on Ruby side, so let's take a look at the database!

# Postgres Database
One reason to move back to FreeBSD was to use jail for database installation. I described how I create jails in one of my previous blog posts. With having the jail ready and being able to access internet, it's time to install Postgres 17 into it by running `pkg -j postgres install postgres17-server`. Notice how I can specify the jail without having to log into it and run the command there using ancient `sh` shell, what a joy!

Let's follow the install notes, enable the service (#1), initialize the database (#2) and start the service (#3):

```
service -j postgres postgresql enable # (1)
service -j postgres postgresql initdb # (2)
service -j postgres postgresql start  # (3)
```

I changed the IP address Postgres listens on by setting `listen_address = 172.16.0.10` in `/var/db/postgres/data17/postgresql.conf` inside the jail. I also allowed the traffic from my laptop into Postgres by adjusting `/var/db/postgres/data17/pg_hba.conf` by adding the following line:

```
host    all             all             172.16.0.1/12           trust
```

Again, feel free to pick whatever IP address you fancy for your jails. Don't forget to restart the service.

The last step is to create 2 databases for my Rails app. This needs to be executed inside the jail:

```
doas jexec postgres createuser --superuser myapp_development
doas jexec postgres createuser --superuser myapp_test
doas jexec postgres createdb myapp_development
doas jexec postgres createdb myapp_test
```

I could also run these commands as SQL commands, set the passwords, etc. But since this is for local development only, I couldn't be bothered. Also, if you thing I called the app `myapp` I must disappoint you, I'm just paranoid to not call it the real name here :-)

# Done!
And that's it! Now I can install gems, create and migrate the database and start hacking my app!

# Links
- [Fixing Ruby gems installation](https://felipec.wordpress.com/2022/08/25/fixing-ruby-gems-installation/)
- [FreeBSD 13: How to Install PostgreSQL (in a Jail)](https://herrbischoff.com/2023/11/freebsd-13-how-to-install-postgresql-in-a-jail/)
- [FreeBSD jails with VNET and NAT](https://www.boucek.me/blog/freebsd-jails-with-vnet-and-nat/)

