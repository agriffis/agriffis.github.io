---
created: 2015-08-12
title: Envy on CoreOS on DigitalOcean
slug: envy
author: Aron
tags: [envy, docker, digitalocean]
layout: article
---

At [PagePart](https://pagepart.com/) we use
[Vagrant](https://www.vagrantup.com/) for development. Our engineers use a mix
of Linux and OS X laptops, and Vagrant allows us to abstract the differences of
those two host platforms by providing a consistent development server
environment with reasonable parity to our production deployment. The result is a
development environment that's easy to set up per dev regardless of their host
environment. The source tree is shared from the host filesystem into the VM so
each dev can se their preferred editor and tools.

For example, on [Fedora](https://getfedora.org/) I use this sequence of steps:

  1. Install VirtualBox from http://rpmfusion.org/
  2. Install Vagrant from http://downloads.vagrantup.com/
  3. `git clone` our repo
  4. `vagrant up` to start the VM
  5. `./run-server.bash` in vagrant to run gunicorn

On OS X, the sequence is exactly the same except for installing VirtualBox which
comes from https://www.virtualbox.org/wiki/Downloads

So this works really well. In fact, it even works to use
[vagrant-lxc](https://github.com/fgrehm/vagrant-lxc) on Linux rather than using
VirtualBox. That makes the VM start up and run a lot faster since it's a
container rather than fully virtualized hardware. The biggest bonus of this
configuration is using bind mounts rather than vboxsf, so filesystem access is
fast and inotify actually works. (The downside is that our provisioning script
has to juggle users and permissions inside the container to match the host.)





for development on my closet-hosted server running Fedora. In point of fact, I'm using vagrant-lxc on KVM rather than bare metal, since that allows me to run a few fully distinct servers on one [Intense PC](http://www.fit-pc.com/web/products/intense-pc/).

This setup works well for most things, but I've recently been thinking about how to improve the development experience 

It works great, but I was recently intrigued by [this screencast](https://vimeo.com/131329120).

  * [Github](https://github.com/progrium/envy)
  * [Getting Started with CoreOS on DigitalOcean](https://www.digitalocean.com/community/tutorial_series/getting-started-with-coreos-2)



Getting 

I was recently intrigued by this screencast: https://vimeo.com/131329120



A few months ago I released two [Bash](http://gnu.org/software/bash)
libraries for parsing command-line options. One is a pure Bash drop-in
replacement for [GNU getopt](http://software.frodo.looijaard.name/getopt/)
called [pure-getopt](http://github.com/agriffis/pure-getopt), since GNU
getopt isn't always available, particularly on OS X. The other library is
called [ghettopt](http://github.com/agriffis/ghettopt), and it's designed
to make command-line parsing much easier than calling getopt manually.

Each of these libraries has a test suite which is designed to be called
from the command-line like this:

    $ bash test.bash

This generates some human-readable output then exits with zero status if
everything passed. Here's an example from ghettopt:

    $ bash test.bash
    (nothing) ... pass
    --foo ... pass
    --foo=x ... pass
    --foo x ... pass
    --foo=x\ y ... pass
    --bar=two ... pass
    --bool ... pass
    --no-bool ... pass
    --bule ... pass
    --no-bule ... pass
    --arr=x --arr=y ... pass
    --help ... pass
    --help= ... pass
    --colon=value ... pass
    x y\ z ... pass
    -- --bool ... pass
    
    16 pass, 0 fail
    
    $ echo $?
    0

By setting the exit status to indicate overall pass/fail, this follows the
pattern of other test suites, and it's easy to plug into [Travis CI](https://travis-ci.org/),
simply creating `.travis.yml` as follows:

    language: bash
    script: bash test.bash

Great, now we're testing on Travis with every push to Github! But what
about testing different versions of Bash? Python and other languages get
explicit
[support for testing with multiple versions](http://about.travis-ci.org/docs/user/build-configuration/#Python)
but there's no provision for Bash. And yet it's important--Bash has changed
dramatically from version to version, especially with the addition of
associative arrays and regular expression matching. My initial
implementations of both pure-getopt and ghettopt weren't backward
compatible to Bash version 2, but there are plenty of older installations,
especially in shared hosting, where Bash 2.05b is still the system default.

Even though Travis doesn't have explicit support for non-system Bash
versions out of the box, it does provide
[package installation using apt](http://about.travis-ci.org/docs/user/build-configuration/#Installing-Packages-Using-apt),
so we can install older Bash versions that way.  In fact, we can also test
newer Bash versions too, since we have no idea what version is installed by
default on the Travis build boxes, and we needn't care.

I've [made a "bashes" PPA available](https://launchpad.net/~agriffis/+archive/bashes).
You can use it with something like this in `.travis.yml`:

    env:
      - BASHES="bash2.05b bash3.0.16 bash3.2.48 bash4.2.45"
    before_install:
      - echo | sudo add-apt-repository ppa:agriffis/bashes
      - sudo apt-get update -qq
      - sudo apt-get install -qq $BASHES

This installs all the available Bash versions simultaneously, which is fine
because the names are versioned, for example `/usr/bin/bash2.05b`.

The trick now is to run the test suite multiple times, once with each
version. One option is to use Travis's
[built-in support for running a matrix of tests](http://about.travis-ci.org/docs/user/build-configuration/#Set-environment-variables),
but I don't recommend that because it instantiates a virtual machine for
each cell in the matrix. Assuming your test suite is small and you don't
anticipate pollution between runs, it's gentler on the Travis
infrastructure to instantiate a single VM and run the suite multiple times
sequentially.

For that we can use a wrapper script which we'll call `travis.bash`. This
relies on the `BASHES` environment variable that we set in `.travis.yml`
above.

    #!/bin/bash
    
    status=0
    
    for b in $BASHES; do
        echo
        echo ==================
        echo $b
        echo ==================
        $b test.bash || status=$?
    done
    
    exit $status

So our final and complete `.travis.yml` contains:

    language: bash
    env:
      - BASHES="bash2.05b bash3.0.16 bash3.2.48 bash4.2.45"
    before_install:
      - echo | sudo add-apt-repository ppa:agriffis/bashes
      - sudo apt-get update -qq
      - sudo apt-get install -qq $BASHES
    script: bash travis.bash

If you use this to test your scripts, please let me know in the comments!