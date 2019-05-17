# Beaker-RSpec

## Glossary

- __DSL__ (Domain-Specific Language): a computer language tailored for specific
  needs. Here, Ruby extensions for testing configuration management systems.
- __SUT__ (System Under Test): the disposable VM or container you are running your
  tests on.


## Directory layout

```
spec/spec_helper_acceptance.rb        # Your common stuff in here
spec/acceptance/myfirsttest_spec.rb   # Your tests here
spec/acceptance/nodesets/default.yml  # Node definitions here
```

## Starting up

- Setup: see the [README](https://github.com/puppetlabs/beaker-rspec)
- Available test primitives: see the
  [ServerSpec docs](https://serverspec.org/).
- A [Puppetlabs real-life example](https://github.com/puppetlabs/puppetlabs-apache/tree/master/spec/acceptance)
  (and more in the Voxpupuli repos). The entry point is in the `.travis.yml`.
- [An example without Travis CI](https://github.com/tequeter/puppet-logcheck/commit/cfdc7db86b989c2153031383ac808519d61c3467).

### Using Beaker-RSpec with PDK

If you are using PDK (and you should), adjust the install steps like this:

Add this to your `.sync.yml` (create the file at the root of your module if it
doesn't exist yet):

```yaml
---
Gemfile:
  required:
    ':system_tests':
      - gem: 'puppet-module-posix-system-r#{minor_version}'
        platforms: ruby
      - gem: beaker
        version: '~> 3.13'
        from_env: BEAKER_VERSION
      - gem: beaker-rspec
        from_env: BEAKER_RSPEC_VERSION
```

Then run `pdk update` to include the new stuff in your `Gemfile`.

You can then run the tests with `pdk bundle exec ...` instead of `bundle exec
...`. That'll install the gems for you as needed.

## Explicit node definitions vs. hostgenerator

The traditional setup is to have a nodeset file describing the nodes needed for
each kind of test you need to run, down to the OS used. This can lead to a lot
of repetition if you are testing for a wide set of OSes.

In the common case where you are running tests on a single node at a time, it
is possible to skip the nodeset definition and provide the important parts as
environment variables instead. In that case, the `beaker-hostgenerator` is
responsible from picking up the OS you are requesting and to make up a nodeset
for it. OTOH, you'll have to pass all the node specification (non-PE, puppet6
etc.) as environment variables.

## CLI

List valid Rake targets:

```
rake -T
```

Run Beaker on nodeset X:

```
BEAKER_debug=true rake beaker:X
```

Run Beaker with a generated single-node nodeset ([source]):

```
PUPPET_INSTALL_TYPE=agent BEAKER_IS_PE=no BEAKER_PUPPET_COLLECTION=puppet6 BEAKER_debug=true BEAKER_setfile=debian9-64{hypervisor=docker} BEAKER_destroy=yes rake beaker
```

(Typically all commands above will be prefixed by `bundle exec` or `pbe`.)

[source]: https://raw.githubusercontent.com/voxpupuli/modulesync_config/master/moduleroot/.github/CONTRIBUTING.md.erb

## How to speed up testing

### How to see real-time output if you are using PDK

See [the Beaker version](testing-beaker.md#how-to-see-real-time-output).

### How to retest without recreating the VM

Set the `BEAKER_destroy=no` and `BEAKER_provision=no` environment variables as
appropriate to inhibit the re-creation and destruction steps.

### How to use a proxy for packages

Like for [the Beaker version](testing-beaker.md#how-to-use-a-proxy), but set
this environment variable instead of the `--package-proxy` arg:

```
BEAKER_PACKAGE_PROXY=http://10.255.0.1:3128
```

## Spec helper

Useful bits for `spec_helper_acceptance.rb`.

### Install Puppet, the module, and the deps

The modern, easy way is:

```ruby
require 'beaker/puppet_install_helper'
require 'beaker/module_install_helper'

run_puppet_install_helper unless ENV['BEAKER_provision'] == 'no'
install_module_dependencies
install_module
```

### RSpec configuration

The strict minimum that everyone uses:

```ruby
RSpec.configure do |c|
  c.formatter = :documentation
end
```

And the [rest](https://www.rubydoc.info/github/rspec/rspec-core/RSpec/Core/Configuration).


### The reusable test that should be in a gem

```ruby
shared_examples 'a idempotent resource' do
  it 'applies with no errors' do
    apply_manifest(pp, catch_failures: true)
  end

  it 'applies a second time without changes' do
    apply_manifest(pp, catch_changes: true)
  end
end
```

This lets you write in your tests:

```ruby
it_behaves_like 'a idempotent resource'
```

