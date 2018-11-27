# Leveraging Powershell DSC with `dsc_lite`

`dsc_lite` is awesome when you want to manage some obscure resource not made
available to Puppet otherwise. But it has its lot of rough edges...

## DSC modules

As a `dsc_lite` user, you rely on a DSC module to get the job done. That module
has to distributed on the target computer (see the `dsc_lite` docs about it),
and bugs in its implementation immediately turn into non-convergent Puppet
behavior (i.e. the same resource changes again and again on every Puppet run).

## Common issues

### Resource ... was not found

This means that Powershell can't load your module, but not necessarily that
you've got the module or resource name in your `dsc_lite` parameters.

Usually you'll get more info by loading the module manually in a Powershell
console:

```powershell
Import-Module <modulename>
```

### Obscure Powershell error

Something went wrong in the DSC module, or you are giving it the wrong
parameters... But all you have as a starting point is an obscure error!

I find it useful to remove the `dsc_lite` layer at this point. Running Puppet
again, this time in `--debug` mode, you can capture the whole Powershell script
that will be used, and the exact commandline invocation. Save the PS script to
a `.ps1` file, and run the commandline from a CMD prompt.

Well-behaved DSC modules should be logging more information in the Verbose
channel. You can enable it by adding the following line at the top of your
`.ps1` file:

```powershell
$script:VerbosePreference = 'Continue'
```

Furthermore, removing the try/catch around `Invoke-DscResource` may also
produce a tad more information when an error happens.

Why not a Powershell prompt? Technically it should work all the same, but I've
grown paranoid to the next issue, that is...

### Module persistence

If you end up modifying the `.psm1` module source (be it to add debug
statements or to fix a bug), you may notice that no matter what you do, your
changes aren't taken into account.

Actually, Powershell keeps modules in memory throughout your Powershell session
(interactive prompt, for example). You have to explicitely unload them to force
a reload:

```powershell
Remove-Module <modulename>
```

... and even then, sometimes it doesn't work. When in doubt, I version my debug
statements to ensure that I'm actually seeing the results of my changes.

Running a powershell invocation in a CMD prompt should be pretty safe from
module persistence, but I'm no Powershell expert.
