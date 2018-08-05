# Beaker: notes for the next adventurer

(Or myself next week.)

## My setup

Linux, PDK 1.6.1, Vagrant, VirtualBox.

As standard as it gets, except that the real standard is running tests on
Puppetlabs's infrastructure. Beyond that, stuff tends to break.


## Useful docs

[Quickstart](https://github.com/puppetlabs/beaker/blob/master/docs/tutorials/quick_start_rake_tasks.md)
to generate sample files to work with and run a first test (Puppet can be
installed). Remember that, if you are using PDK, you should include the
Rakefile change in your `.sync.yml` instead:

```yaml
---
Rakefile:
  requires:
    - beaker/tasks/quick_start
```

[The Beaker DSL](https://github.com/puppetlabs/beaker/blob/master/docs/how_to/the_beaker_dsl.md).
The most useful bit in the docs, probably. Err, scrape that: most of the links
are obsolete :(

[Beaker::TestCase](https://www.rubydoc.info/github/puppetlabs/beaker/Beaker/TestCase).
Your tests and pre/post/whatever suites extend that class, so all the
methods and properties listed there are available immediately.

There are much more docs on the beaker repo, but so far RTFSing has been more
useful in my endeavors.


## Directory layout

As created by the quickstart task,

```
acceptance/config/  # Your node definition files go here (*.yaml)
acceptance/setup/   # Your pre-test suites go here (*.rb)
acceptance/tests/   # And the actual tests here (*.rb)
```

## How to speed up testing

Learning Beaker can be highly frustrating because of each attempt takes minutes
to spit out the next error message (if you actually made progress, that is ;).

We need to fail faster.

### How to see real-time output

`pdk bundle exec ...` buffers the output, e.g. you won't see the result of your
test until it's finished. It's a
[known limitation](https://github.com/puppetlabs/pdk/issues/364).

My trivial workaround: I created a script (I'll call it `pbe`) that just calls
`bundle` with the right environment variables set:

```bash
#!/bin/sh

PUPPET_GEM_VERSION=5.5.3 BUNDLE_IGNORE_CONFIG=1 GEM_HOME=... GEM_PATH=... PATH=... \
    /opt/puppetlabs/pdk/private/ruby/2.4.4/bin/bundle exec "$@"
```

You can get the values for these environment variables by running `pdk --debug
bundle exec`.

NB: you'll need to run `pdk bundle exec` at least once to install any new gem
referenced in the `Gemfile`.

### How to retest without recreating the VM

The official way: use
[subcommands](https://github.com/puppetlabs/beaker/blob/master/docs/tutorials/subcommands.md).

Of course, it [doesn't
work](https://tickets.puppetlabs.com/projects/BKR/issues/BKR-1469) anymore with
Vagrant.

My workaround: if you specify `--preserve-hosts=always`, the SUTs won't be
destroyed and you'll even get the exact command line to run to retest under the
`You can re-run commands against` line. It assumes that you have
the environment variables set, so I just use my `pbe` script.

But wait! Of course it can't be that easy, there is one more Vagrant-related
bug. You'll get:

```
#<RuntimeError: Beaker is configured with provision = false but no vagrant file
was found at
...module.../.vagrant/beaker_vagrant_files/hosts_preserved.yml/Vagrantfile.
You need to enable provision>
```

The Vagrantfile is actually in a directory called `default.yaml`, after the
name of the hosts definition file (so adapt this as needed). I didn't bother to
google this one up and just created a symlink:

```bash
cd .vagrant/beaker_vagrant_files
ln -s default.yaml hosts_preserved.yml
```

### How to use a proxy

Downloading large packages (like the puppet-agent bundle) on a crappy ADSL
takes ages. Surely we could cache them?

Install a Squid (or whatever other cache) on your workstation, do the minimal
configuration to allow access from your `vboxnet2` network (not the whole
Internet :).

Then just pass the argument `--package-proxy http://10.255.0.1:3128` (or
whatever is the VirtualBox-facing IP of your Squid).

This one was easy and actually worked out of the box!

The only caveat is that Vagrant runs apt-get update and installs rsync before
Beaker can configure the proxy, so it's still not as fast as it could be.


## How to call more Ruby

I didn't notice at once that Beaker was configuring the proxy by itself, so
I figured out the following snippet before I realized my mistake.

I'm documenting it here, no doubt it'll be useful later ;)

```ruby
require 'beaker/host_prebuilt_steps'
extend Beaker::HostPrebuiltSteps

hosts.each do | h |
  package_proxy(h, options)
end
```
