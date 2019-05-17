# Testing Puppet Code

## Intro

### Why test?

The degree zero of testing is just applying the code on a real node. Hopefully
not production, and hopefully with a `--noop` first.

That works at first, but it quickly gets old to reapply your code on your
growing number of representative nodes every time you make a potentially
breaking change.

There's got to be a better way, right?

### How much testing is good testing?

In my experience, software testing is a diminishing returns game.

In Puppet, there are some tests that take nearly no effort to write that will
already help tremendously compared to no testing at all.

The more tests you add however, the more you have to maintain besides your
actual value-delivering code.

The exact sweet-spot will depend on the project and the complexity.

It's probably a good idea to add tests when reporting/fixing bugs. That part of
the code wasn't so trivial that it could go without tests, after all :)


## Testing tools

Puppet modules can typically be tested with:

- Validation tools for syntax and format.
- RSpec for unit testing.
- Beaker-RSpec for actual deployment on a single node.
- Beaker for multi-node deployment.


### Validation tools

That's the easiest test. Just run from your module's root directory:

```
pdk validate
```

(You're using [PDK](https://puppet.com/docs/pdk/1.x/pdk.html), right?)

That'll get you these checks for free:

- Puppet syntax and style (Puppetlabs' Coding Style).
- JSON syntax and style for `metadata.json` and tasks.
- Ruby style for your Ruby extensions and Puppet tests.

Your IDE might already implement some of these checks in real time.

### RSpec

RSpec compiles a [Puppet
catalog](https://puppet.com/docs/puppet/5.5/subsystem_catalog_compilation.html)
with part of your code and tests the contents of the
catalog. Your code is never actually applied on a node, but there is already a
surprisingly high amount of bugs to be found this way.

By creating classes, defined types etc. with the following command, you'll get
the most basic check for free: that the code compiles.

```
pdk new class mymodule::myclass
```

RSpec tests live under `spec/`.

RSpec is quite easy because it's well documented. The main learning curve is
the RSpec mini-language (DSL) itself.
The [Wikipedia page](https://en.wikipedia.org/wiki/RSpec) has the basic intro,
then switch to the [Puppet specifics](http://rspec-puppet.com/).

### Beaker-RSpec

Beaker-RSpec tests (not to be confused with Beaker) actually deploy a node with
your code. That's when you find out that the configuration file had all the
entries you expected, but the app you're Puppetizing doesn't like them :)

This part is still easy to work with once you figure out the basics.

All the tests and data live under `spec/acceptance/`.

The DSL is identical to the one of the RSpec tests above, so that's one less
thing to learn. The setup step is decently documented in the
[README](https://github.com/puppetlabs/beaker-rspec) and the available tests
primitives in the [ServerSpec docs](https://serverspec.org/).

It is also widely used in the `puppetlabs/*` and `puppet/*` modules, so it's
easy to find
[examples](https://github.com/puppetlabs/puppetlabs-apache/tree/master/spec/acceptance).
The tests being run by Travis CI, the entry point of these examples is in
`.travis.yml`, along with the environment variables that are part of the
configuration.

More information on [my Beaker-RSpec page](testing-beaker-rspec.md).

### Beaker

If you wish to test situations that Beaker-RSpec can't cope with, you'll need
the full-blown Beaker:

- Testing several nodes at once (cluster behavior, client/server etc.).
- Rebooting the nodes.

And... Here Be Dragons. Sorry.

The [docs](https://github.com/puppetlabs/beaker/tree/master/docs) are 50%
obsolete, 50% a mess, and 100% not useful to your usecases.

Actually, unless you are working for Puppetlabs with access to the experts and
running your tests on their infrastructure, you're in for a bumpy ride.

I'll document my findings in a [separate file](testing-beaker.md).

The Beaker tests live in `acceptance/`, not `spec/acceptance/`.
