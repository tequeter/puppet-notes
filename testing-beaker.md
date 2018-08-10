# Beaker: notes for the next adventurer

(Or myself next week.)

## My setup

Linux, PDK 1.6.1, Vagrant, VirtualBox.

As standard as it gets, except that the real standard is running tests on
Puppetlabs's infrastructure. Beyond that, stuff tends to break.


## Useful docs

[Quickstart] to generate sample files to work with and run a first test
(install Puppet and try to run it). Remember that, if you are using PDK, you
should include the Rakefile change in your `.sync.yml` instead:

```yaml
---
Rakefile:
  requires:
    - beaker/tasks/quick_start
```

[The Beaker DSL], as a list of useful functions. Some/most of links are broken
because of the [4.0 release] which moved most of these methods to [separate
modules], most importantly [beaker-puppet].

[Beaker::TestCase]. Your tests and pre/post/whatever suites extend that class,
so all the methods and properties listed there are available immediately.

[Style Guide], some useful tips in there.

There are much more docs on the beaker repo, but so far RTFSing has been more
useful in my endeavors.

[Quickstart]: https://github.com/puppetlabs/beaker/blob/master/docs/tutorials/quick_start_rake_tasks.md
[The Beaker DSL]: https://github.com/puppetlabs/beaker/blob/master/docs/how_to/the_beaker_dsl.md
[4.0 release]: https://github.com/puppetlabs/beaker/blob/4.0.0/docs/how_to/upgrade_from_3_to_4.md
[separate modules]: https://www.rubydoc.info/find/github?q=beaker
[beaker-puppet]: https://www.rubydoc.info/github/puppetlabs/beaker-puppet
[Beaker::TestCase]: https://www.rubydoc.info/github/puppetlabs/beaker/Beaker/TestCase
[Style Guide]: https://github.com/puppetlabs/beaker/blob/master/docs/concepts/style_guide.md


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


## Setup suite tips

If you recall from the Quickstart in the official documentation, the setup
suite is where you get all the prerequisites installed on the SUTs.

### Installing Puppet 5

Getting the latest version of Puppet installed on the SUT is easy, but
non-obvious:

```ruby
install_puppet_agent_on(hosts, puppet_collection: 'puppet5')
```

### Installing your module on the SUT

Just as easy, but:

- Your setup file must have `_spec` in the name (for example,
  `acceptance/setup/puppet_and_module_spec.rb`).
- Remember to declare the `beaker-module_install_helper` gem in your
  `.sync.yml`, `Gemfile` section.
- Of course, your `metadata.json` must be up to date WRT dependencies (you'll
  find out quickly if it isn't :).

```ruby
require 'beaker/module_install_helper'

install_module
install_module_dependencies
```

## Test tips

### Testing your module

Some useful methods to test Puppet code:

#### `apply_manifest_on`

```ruby
apply_manifest_on(hosts,
                  'class { "yourmodule": args... }',
                  expect_changes: true)
```

This runs some Puppet code, such as calling the entry points of your module
with the right arguments.

Call it once with `expect_changes` set to true (your module should "do stuff",
on a pristine system), and a second time with that argument set to false. After
installing/configuring whatever your module manages, the SUT should be in the
desired state and further Puppet runs should not have to modify anything.

[Documentation](https://www.rubydoc.info/github/puppetlabs/beaker-puppet/Beaker/DSL/Helpers/PuppetHelpers#apply_manifest_on-instance_method).


### Declare teardowns

If you intend to recycle VMs to speed up testing, it's critical to declare the
necessary steps to bring back the SUT in a state suitable for testing again, no
matter where your test actually stops.

These [teardown blocks] go _before_ the steps they clean up:

```ruby
test_name 'check /some/file gets installed' do
  teardown do
    on(hosts, 'rm -f /some/file')
  end

  step 'install /some/file' do
    apply_manifest_on(hosts, ...)
  end
end
```

[teardown blocks]: https://github.com/puppetlabs/beaker/blob/master/docs/concepts/style_guide.md#teardowns

### Calling Beaker-RSpec

Getting the best of both worlds would be super convenient, but I have no idea
how, so far :)
