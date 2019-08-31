# Beaker: notes for the next adventurer

(Or myself next week.)

## Glossary

- __DSL__ (Domain-Specific Language): a computer language tailored for specific
  needs. Here, Ruby extensions for testing configuration management systems.
- __SUT__ (System Under Test): the disposable VM or container you are running your
  tests on.

## This is not about Beaker-RSpec

Don't bother with what follows if Beaker-RSpec can fit the bill for you: you
don't need to reboot your SUTs mid-test, and you don't need more than one SUT
at once.

## My setup

Linux, PDK 1.12.0, Vagrant, VirtualBox.

As standard as it gets, except that the real standard is running tests on
Puppetlabs's infrastructure. Beyond that, stuff tends to break.


## Useful docs

### Introductory / concepts

[Quickstart] to generate sample files to work with and run a first test
(install Puppet and try to run it). Remember that, if you are using PDK, you
should include the Rakefile change in your `.sync.yml` instead:

```yaml
---
Rakefile:
  requires:
    - beaker/tasks/quick_start
```

What a [test should look like]. I wish this had been easier to find...

[The Beaker DSL], as a list of useful functions. Some/most of links are broken
because of the [4.0 release] which moved most of these methods to [separate
modules], most importantly [beaker-puppet].

[Style Guide], some useful tips in there.

There are much more docs on the beaker repo, but so far RTFSing has been more
useful in my endeavors.

[Quickstart]: https://github.com/puppetlabs/beaker/blob/master/docs/tutorials/quick_start_rake_tasks.md
[The Beaker DSL]: https://github.com/puppetlabs/beaker/blob/master/docs/how_to/the_beaker_dsl.md
[4.0 release]: https://github.com/puppetlabs/beaker/blob/4.0.0/docs/how_to/upgrade_from_3_to_4.md
[separate modules]: https://www.rubydoc.info/find/github?q=beaker
[beaker-puppet]: https://www.rubydoc.info/github/puppetlabs/beaker-puppet
[Style Guide]: https://github.com/puppetlabs/beaker/blob/master/docs/concepts/style_guide.md
[test should look like]: https://www.rubydoc.info/github/puppetlabs/beaker/Beaker/DSL/Structure

### API

[Beaker::TestCase]. Your tests and pre/post/whatever suites extend that class,
so all the methods and properties listed there are available immediately.

There are much more useful methods in the [Beaker::DSL] and the classes in that
namespace (don't miss them). And then there are all the [beaker-puppet] DSL
extensions, just as important as the main gem for our purposes.

[Unix::Host], that `host` object, even for Windows SUTs. NB: it also
has properties you can lookup, but I haven't found a reference for these.

When actually testing stuff, you'll want to assert the results with [Minitest],
Ruby's syntax for test suites.

[Beaker::TestCase]: https://www.rubydoc.info/github/puppetlabs/beaker/Beaker/TestCase
[Beaker::DSL]: https://www.rubydoc.info/github/puppetlabs/beaker/Beaker/DSL
[beaker-puppet]: https://www.rubydoc.info/github/puppetlabs/beaker-puppet/
[Unix::Host]: https://www.rubydoc.info/github/puppetlabs/beaker/Unix/Host
[Minitest]: http://docs.seattlerb.org/minitest/Minitest/Assertions.html


## Directory layout

As created by the quickstart task,

```
acceptance/config/  # Your node definition files go here (*.yaml)
acceptance/setup/   # Your pre-test suites go here (*.rb)
acceptance/tests/   # And the actual tests here (*.rb)
```

## Basic Usage

Beaker can be made to work through PDK (`pdk bundle exec`), but this is mostly
self-inflicted pain. I've updated the instructions below to use Bundler
instead.

### Required Gems

Your `.sync.yml` (for PDK) should include the following:

```yaml
Gemfile:
  required:
    ':system_tests':
      - gem: beaker
        version: '~> 4.8'
        from_env: BEAKER_VERSION
      - gem: beaker-module_install_helper
      - gem: beaker-puppet
      - gem: beaker-vagrant
        version: '~> 0.6'
        from_env: BEAKER_VAGRANT_VERSION
```

Then get a working Bundler environment with:

```
bundle install --path vendor/ --with system_tests --jobs 4`
```

It is important to use `vendor` instead of `.vendor`, as PDK ignores the former
but not the later, and would otherwise report lots of YAML errors in files that
don't belong to you.

Adjust `--jobs` to your number of CPU cores.

### Option Files and Rake

This avoids having to copy-paste standard arguments over and over.

Assuming we're going to create a "run everything" test suite, start by creating
a Beaker option file as `acceptance/.beaker-everything.cfg`:

```ruby
{
  :pre_suite       => ['./acceptance/setup'],
  :hosts_file      => './acceptance/config/default.yaml',
  :log_level       => 'debug',
  :tests           => ['./acceptance/tests'],
  :timeout         => 6000
}
```

If you also add the Beaker/Rake integration (here in PDK `.sync.yml` format):

```yaml
Rakefile:
  requires:
    - require: beaker/tasks/test
      conditional: "Bundler.rubygems.find_name('beaker').any?"
```

Then you can invoke your "everything" acceptance test suite with `bundler exec
rake 'beaker:test[,everything]'`, or as `bundler exec beaker --options-file
./acceptance/.beaker-everything.cfg`.

The second form lets you pass extra Beaker arguments on the command line, but
anything that you use every time should go in the `.cfg`.


## How to speed up testing

Learning Beaker can be highly frustrating because of each attempt takes minutes
to spit out the next error message (if you actually made progress, that is ;).

We need to fail faster.

### How to retest without recreating the VM

The official way,
[subcommands](https://github.com/puppetlabs/beaker/blob/master/docs/tutorials/subcommands.md),
now works for Vagrant too with `beaker-vagrant >= 0.6` and a couple of patches
to the `beaker` and `beaker-vagrant` gems!

You can run:

```
bundle exec beaker init -h acceptance/config/default.yaml -o acceptance/.beaker-everything.cfg --any-other-option
bundle exec beaker provision
bundle exec beaker exec setup,tests
# Fix your tests here
bundle exec beaker exec tests
# More iterations
bundle exec detroy
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
