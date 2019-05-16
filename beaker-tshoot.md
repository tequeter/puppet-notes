# Troubleshooting testing with Beaker / Beaker-RSpec

Beaker has a lot of rough edges, even more so on Windows (the typical workstation at my customers).

## Errors and solutions

### Vagrant fails with `stdin is not a tty` (Windows)

I had this error with Vagrant 1.9.7. Upgrading to Vagrant 2.1.4 solved it.

The error message is emitted by `winpty` in the Vagrant launcher, FWIW.

### VirtualBox, hostonly interfaces, and administrator privileges

VirtualBox will cause a bunch of administrator prompts to create the host-only interface,
but it seems to stop after a first succesful startup.

### Package curl cannot be queried on vmname

In your `spec/acceptance/nodesets/xxx.yml`, you specified an invalid platform, like `redhat-7-x64` instead of `el-7-x64`.

### Can't reconnect to an existing SUT for Beaker-RSpec

If you tried to speed up your testing by following the [documented workflow],
setting `BEAKER_provision="no"` and/or `BEAKER_destroy="no"` you've met this error after a long delay:

```
Warning: Skipping ip method to ssh to host as its value is not set. Refer to https://github.com/puppetlabs/beaker/tree/master/docs/how_to/ssh_connection_preference.md to remove this warning
Warning: Skipping vmhostname method to ssh to host as its value is not set. Refer to https://github.com/puppetlabs/beaker/tree/master/docs/how_to/ssh_connection_preference.md to remove this warning
Warning: Try 1 -- Host yourvmname unreachable: SocketError - getaddrinfo: No such host is known.
Warning: Trying again in 3 seconds
...
```

I haven't found a workaround for Beaker-RSpec like I did for [Beaker][2].
So for now you'll have to recreate the VM every time. Yay.

The [bug] is known since 2015. Go vote for it! (or fix it if you can...)

[documented workflow]: https://github.com/puppetlabs/beaker-rspec/blob/master/README.md
[2]: testing-beaker.md#how-to-retest-without-recreating-the-vm
[bug]: https://tickets.puppetlabs.com/browse/BKR-494

### Metadata.json and unpublished modules

`beaker/module_install_helper`'s `install_module_dependencies_on()` rely on your `metadata.json`
to install the dependencies, and expect all of them to exist on the Forge.

This is going to be a problem when you have dependencies that are:

1. Public modules not published on the Forge (or a fork pending PR approval).
2. Unpublished versions of public modules (PR was merged but no new Forge release).
3. Internal (customer-proprietary) modules.

There is no general solution unfortunately, because the world (Puppetlabs, Vox Pupuli, ...)
only deals with very few, public, dependencies, and/or has their private Forge.

If you are trying to test a role or profile (which would explain the dependencies),
[Onceover] seems a good candidate for a solution (NB: I didn't test this yet).

[Onceover]: https://github.com/dylanratcliffe/onceover

Otherwise, you'll have to resort to Beaker-level hacks in your `spec/spec_helper_acceptance.rb`
to pull all the dependencies from the 'Net, even those that `install_module_dependencies_on()`
would have successfully found.

Indeed, `module_install_helper` is going to choke on anything not found in the Forge (module or version).
Even a filtering approach is going to fail, because the version querying happens before your filter:

```ruby
deps = module_dependencies_from_metadata       # FAILS HERE
forge_deps = deps.reject { |item| item[:module_name] =~ %r{^site-|unpublishedmodule} }
install_module_dependencies_on(hosts, forge_deps)
```

... aaand a flat-out wrong but Forge-friendly `metadata.json` is going to cause other havoc, as
the Puppet parser relies on metadata.json to find type aliases. That's going to fail your unit tests.

Here is a sample hack in `spec/spec_helper_acceptance.rb` using r10k for a profile.
Subversion is required by the repositories in the Puppetfile.
The whole thing could probably be happily replaced by Onceover.

```ruby
hosts.each do |host|
  modulesdir = "#{host.puppet['codedir']}/modules"
  scp_to(host, File.join('..', '..', 'Puppetfile'), modulesdir)
  on host, '/opt/puppetlabs/puppet/bin/gem install r10k'
  #on host, '<site-specific RHN registration, if necessary>'
  apply_manifest_on(host, 'package { "subversion": ensure => "installed" }', catch_failures: true)
  on host, "cd #{modulesdir} && /opt/puppetlabs/puppet/bin/r10k puppetfile install"
end
```

For a technological module with few, public, dependencies, you can probably get away with hardcoding
the git calls there until the modules/versions are published on the Forge and you can revert to
`module_install_helper`.

### Beaker dependencies on Windows

Acceptance (Beaker / Beaker-RSpec) tests make use of a nifty meta-gem that just pulls in a bunch of useful
gems. You just have to figure out how to use it: on Windows, copy-pasting the boilerplate won't cut it.

Here is what should be in your `.sync.yml` to support both POSIX and Windows Beaker workstations, AFAIK:

```yaml
      - gem: 'puppet-module-posix-system-r#{minor_version}'
        platforms: ruby
      - gem: 'puppet-module-win-system-r#{minor_version}'
        platforms: x64_mingw
```

The first gem is found in most modules, but the `ruby` platform actually restricts it to POSIX platforms.
The second one is required for Windows systems, but isn't widely known.

You may have to allow it to be used on more platforms than just `x64_mingw` (that one works for the PDK at least),
see the [platform list].

[platform list]: https://bundler.io/man/gemfile.5.html#PLATFORMS

## Troubleshooting techniques

### Vagrant environment variables

### Beaker environment variables

* `BEAKER_destroy`
* `BEAKER_debug`
