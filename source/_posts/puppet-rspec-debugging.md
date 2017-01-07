---
title:  puppet-rspec debugging
date: 2012-04-03
---
While introducing [rspec-puppet](https://github.com/rodjek/rspec-puppet) into a big and grown puppet codebase at Jimdo we needed to debug stuff and get more verbose output while writing the first tests. As the interwebs aren't very chatty about the topic, here for all the distressed googlers:

Configure debug output and a console logger in your test (or helper or somewhere):

    it "should do stuff" do
      Puppet::Util::Log.level = :debug
      Puppet::Util::Log.newdestination(:console)
      should ...
    end
    

hth :)