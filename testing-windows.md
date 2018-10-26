# Testing on Windows

## PDK

The easy way is just to have PDK-compatible modules, then you can:

  pdk validate
  pdk test unit
  pdk bundle exec rspec spec\acceptance
  pdk bundle exec rake ...

## Non-PDK Modules

You need a separate Ruby install, then from a Windows shell in the module you
want to test, run:

  c:\Ruby24-x64\bin\bundle install

This will install all dependencies in `bundle`. It's better (IMHO) than
installing all gems from all modules you might contribute to in your Ruby's
install, as would `gem install ...` do. Furthermore, this may work around
permission issues (and letting all users write to the Ruby install doesn't seem
very secure in the first place).

But I'm no Ruby wiz.

Equivalent commands for the above:

  c:\Ruby24-x64\bin\rubocop
  c:\Ruby24-x64\bin\bundle exec rake spec
  c:\Ruby24-x64\bin\bundle exec rake ???
  c:\Ruby24-x64\bin\bundle exec rake ...

(Calling `bundle exec rake rubocop` insists on inspecting all the gems under
`bundle`...)
