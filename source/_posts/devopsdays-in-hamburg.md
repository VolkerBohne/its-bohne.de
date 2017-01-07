---
title:  devopsdays in Hamburg
date: 2010-10-17
---
the last weekend i attended to **[devopsdays](http://www.devopsdays.org/)** in hamburg. first thanks to **[patrick debois](http://twitter.com/#!/patrickdebois)** and **[marcel wegermann](http://pc0fc2375091e8a53)** for doing a really great job of conference organisation. and thanks to the sponsors making the location, food and beer possible ;)

[DevOps](http://en.wikipedia.org/wiki/DevOps) is a relatively young movement of people that think developers, operations and also QA have to work together instead of
creating departments and isles. It's about **communication** and **automation of software delivery processes**.

For example [Jez Humble](http://twitter.com/jezhumble) had a talk on "[Continuous Delivery](http://continuousdelivery.com/)" (the book is must read) which is the brother of Continuous Integration: If you integrate continuous why not deploy it continuously. **Rule 1: DONE MEANS RELEASED**

How does that fit with Scrum? Scrum says: DONE MEANS "PASSES ACCEPTANCE TESTS" and will be released "somewhen" (well, yes, this is my very personal problem with scrum ;) )

So there were talks about and even a real-world example introduction of **Kanban **which IMHO perfectly fits the continuous delivery process. Check out this nice [Scrum vs. Kanban minibook](http://www.crisp.se/henrik.kniberg/Kanban-vs-Scrum.pdf). In kanban you can visualize your **[value chain](http://en.wikipedia.org/wiki/Value_chain)** - and in software development a value is commonly created at the time you deployed / released your changes so it's being used by your customers / users - so follows **RULE 2: "ready for blahblub" is waste.**

So we want our changes deployed continuously and automatically, but we do not want to press the red button until we are sure that production will not be destroyed. so we need unit tests, functional tests and so on. this is what jez's book is all about. i also heard about nice tools like [Vagrant](http://vagrantup.com/), which allows you to simulate production env on every developer's laptop with VirtualBox and integrating it into your configration management. Or firing up some instances for integration / functional testing etc (very short and incomplete explanation, yeah i know)

Ah configuration management. Didn't get any? How many servers to manage? Does not matter. **RULE 3: USE CONFIG MANAGEMENT OR DIE** (more or less rule zero). Use [puppet](http://www.puppetlabs.com/) or [chef](http://wiki.opscode.com/display/chef/Home). those were the most mentioned tools at the conf.

As mentioned, Devops is also about communication and evangelism. Th﻿ere was some thinking of a **[devops manifesto](http://theagileadmin.com/2010/10/15/a-devops-manifesto/)** like the agile one.

Just another rule: I had some small discussion with Jez about feature branches. Feature branches and cont. integration do not well fit together. Rather use **[branch by abstraction](http://paulhammant.com/blog/branch_by_abstraction.html).**

This are some just my very own impressions and learnings of this weekend, there was even much more stuff. Hopefully videos will come online in the near future :)

Thanks all people involved. I really appreciate the devops movement. Feels like coming from heaven at the right time.
