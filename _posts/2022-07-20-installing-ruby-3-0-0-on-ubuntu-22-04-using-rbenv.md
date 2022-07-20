---
layout: post
title: Installing ruby-3.0.0 on Ubuntu 22.04 using rbenv
date: 2022-07-20 15:22 -0400
tags: ruby
---
*TL;DR: If you are in a hurry then refer to the [Commands](http://127.0.0.1:4000/posts/installing-ruby-3-0-0-on-ubuntu-22-04-using-rbenv/#commands) section below.*

## Summary

This blog was created using [jekyll](https://jekyllrb.com/) a static site generator. I won't go into any more details than that for now, but I did want to quickly mention a problem I encountered as part of my development workflow. Jekyll relies on ruby and to install ruby I used [rbenv](https://github.com/rbenv/rbenv) which is a ruby version manager. When I first started building my blog, I used ruby version 3.0.0 and at the time it installed without issue on my laptop running Ubuntu 20.04.

## Problem

However, that is not my main computer, and I usually find myself switching between my Macbook pro and my Windows gaming pc. On Windows, I use WSL for my development needs and have multiple distros installed. I recently started using Ubuntu 22.04 and that's where I encountered the following problem. As part of my setup of the new distro, I installed rbenv and tried to install ruby-3.0.0 but was halted by this error message:

```shell
$ rbenv install 3.0.0
Downloading ruby-3.0.0.tar.gz...
-> https://cache.ruby-lang.org/pub/ruby/3.0/ruby-3.0.0.tar.gz
Installing ruby-3.0.0...

BUILD FAILED (Ubuntu 22.04 using ruby-build 20220630)

Inspect or clean up the working tree at /tmp/ruby-build.20220720133848.5660.mjaF0x
Results logged to /tmp/ruby-build.20220720133848.5660.log

Last 10 log lines:
make[2]: Leaving directory '/tmp/ruby-build.20220720133848.5660.mjaF0x/ruby-3.0.0/ext/date'
cc1: note: unrecognized command-line option ‘-Wno-self-assign’ may have been intended to silence earlier diagnostics
cc1: note: unrecognized command-line option ‘-Wno-parentheses-equality’ may have been intended to silence earlier diagnostics
cc1: note: unrecognized command-line option ‘-Wno-constant-logical-operand’ may have been intended to silence earlier diagnostics
linking shared-object socket.so
make[2]: Leaving directory '/tmp/ruby-build.20220720133848.5660.mjaF0x/ruby-3.0.0/ext/socket'
linking shared-object ripper.so
make[2]: Leaving directory '/tmp/ruby-build.20220720133848.5660.mjaF0x/ruby-3.0.0/ext/ripper'
make[1]: Leaving directory '/tmp/ruby-build.20220720133848.5660.mjaF0x/ruby-3.0.0'
make: *** [uncommon.mk:300: build-ext] Error 2
```

## Solution

After an hour of google searches I discovered my problem - It seems ruby-3.0.0 was not compatible with the OpenSSL version installed on Ubuntu 22.04 (version 3.0). There were several github threads that offered solutions, mainly, compile an older version of OpenSSL into its own directory on your machine and then point to that directory as when you run the `rbenv install` command.

At first glance, it seem like a lot of effort  to install the version of ruby I wanted. I tested updating my Jekyll deployment to a higher version of ruby that was compatible with Ubuntu 22.04 (version 3.1.1), and that worked fine, but for whatever reason I never went forward with it. Fast forward a few days, when I finally decided to compile OpenSSL myself and I'm glad I did because the process was super easy. Most of it was spent waiting for commands to finish. As outlined by a github user [here](https://github.com/rbenv/ruby-build/discussions/1940#discussioncomment-2663209), the following commands are all it takes to compile a sperate version of OpenSSL:

### Commands

```shell
# install the dependencies
sudo apt install build-essential checkinstall zlib1g-dev

# download an older release from here https://www.openssl.org/source/old/ (e.g. 1.1.1)
wget https://www.openssl.org/source/old/1.1.1/openssl-1.1.1p.tar.gz

# extrat contents of the file
tar -xvf openssl-1.1.1p.tar.gz

# cd into the directory of the file contents and compile it with these steps
./config --prefix=/opt/openssl-1.1.1q --openssldir=/opt/openssl-1.1.1q shared zlib
make
make test
sudo make install

# link the system's certs to OpenSSL 1.1.1 directory
sudo rm -rf /opt/openssl-1.1.1q/certs
sudo ln -s /etc/ssl/certs /opt/openssl-1.1.1q

# install ruby using 'RUBY_CONFIGURE_OPTS=--with-openssl-dir=/opt/openssl-1.1.1q' before the command
RUBY_CONFIGURE_OPTS=--with-openssl-dir=/opt/openssl-1.1.1q rbenv install 3.0.0

# to make this permanent, update .zshrc with this:
export RUBY_CONFIGURE_OPTS="--with-openssl-dir=/opt/openssl-1.1.1q/"
```

And that's it! Assuming all goes well, your version of ruby will have installed successfully and you can go on with your day.
