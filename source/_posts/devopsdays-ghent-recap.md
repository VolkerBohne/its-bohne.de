---
title:  devopsdays Ghent recap
date: 2014-11-02
---

**!!! ATTENTION: Highly unstructured braindump content !!!**

### Day 1

#### [The Self-Steering Organization: From Cybernetics to DevOps and Beyond](http://www.slideshare.net/ingineeringit/from-cybernetics-to-devops-and-beyond)

Nice intro intro cybernetics and systems theory. Nothing really new for me as I'm into system theory _a very little_ bit. Keywords: Auto autopoiesis, systems theory, cybernetics,  empathy, feedback.

#### [Ceci n'est pas \#devops](http://bridgetkromhout.com/speaking/2014/devopsdays-belgium/)

* "DevOps is culture, anyone who says differently is selling something. Tools are necessary but not sufficient. Talking about DevOps is not DevOps."
* fun experiment replace every occurrence of "DevOps" with "empathy" and see what happens ;-) reminded me of the ["butt plugin"](https://chrome.google.com/webstore/detail/cloud-to-butt-plus/apmlngnhgbnjpajelfkmabhkfapgnoai?hl=en))

#### [Cognitive Biases in Tech: Awareness of our own bugs in decision making](https://speakerdeck.com/nigelkersten/cognitive-biases-in-tech-awareness-of-our-own-bugs-in-decision-making)

* Talk is mainly backed by [the book "Thinking, fast and slow"](http://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow)
* Brain is divided in System 1 and System 2
* System 2 gets tired easily: Do important decisions in the morning (e. g. monolith vs. micro-service), postpone trivial ones to the evening (e. g. what to cook for dinner)
* great hints for better post mortems

#### [5 years of metrics and monitoring](https://speakerdeck.com/auxesis/5-years-of-metrics-and-monitoring)

* [great recap on infoq](http://www.infoq.com/news/2014/10/5-years-metrics-monitoring)
* You really have to focus on how to visualize stuff. Looks there needs to be expertise for this in a company which wants to call itself "metrics driven" or "data driven"
* We have to be aware of Alert fatuigues:
  * noise vs. signal
  * not reacting to alerts anymore, because "they will self-heal anyway in a few minutes" (we call this "troll-alert" internally, which is a very bad description for an alert coming from a non-human system which is apparently not able to troll)

#### Ignites

#### Repository as an deployment artifact - Inny So

* talking about apt.ly - application+environment as atomic release tags

### Day 2

#### Running a fully self-organizing/self-managing team (or company)

* [good recap at infoq](http://www.infoq.com/news/2014/10/devops-days-belgium-2)
* interesting open allocation work model, but with managers, feedback loops, retrospectives and planning meetings. They call it "self-selection"
* it's sometimes better to let people go instead of trying to keep them
* people need explicit constraints to work in, otherwise they don't know their and others boundaries

#### [Automation with humans in mind: making complex
systems predictable, reliable and humane](https://gist.github.com/s0enke/0ac2f6a0cce307d9cddc)

* [InfoQ](http://www.infoq.com/news/2014/10/devops-days-automation-humans) ;-)

#### Open spaces

##### Internal PaaS

I hosted a session on "Why/How to build an internal PaaS". The reason for doing this is building a foundation for (micro-)services: Feature Teams should be able to easily deploy new services (time to market <1hour). They should not care about building their own infrastructure for: Deployment of appliations of different languages (PHP, Ruby, Java, Python, Go ...), metrics, monitoring, databases, caches etcpp.

So I had a quick look, e.g. at flynn.io or deis.io, which pretend to do what I want, and I hoped someone actually using stuff like that might be right here.

The session itself was a bit clumsy: I guess I couldn't explain my problem well enough, or it actually is no problem. Or the problem is too new as there was no one in the room who actually had more than 2 micro-services deployed.

But anyway, talking about what I want to achieve actually helped me to shape my thoughts.

##### Microservices - What is important for Ops

* Session hosted by [MBS](https://twitter.com/bruntonspall)
* If a company wants to migrate to / implement microservices, an Ops team should insist on 12-factor-app style in order to have a standardization
* Have a look at Simian Army which has been implemented to mitigate common bad practices in microservice architecture, e. g. make everyone aware of fallacies of distributed computing.
* Not really for Ops, but for devs:
  * EBI / Hexagonal programming style from beginning on, so it doesnt matter (in theory) if monolithic or service-oriented. In theory easy to switch
  * Jeff Bezos Rules
  * Generally having a look at Domain Driven Design and orienting (e. g. using repository and entities instead of ActiveRecord)

### All the videos

[on ustream](http://www.ustream.tv/recorded/54693964)

### Other Recaps

* [HangOps](http://t.co/P5iyFIZKnL)
* [InfoQ Day 1](https://gist.github.com/s0enke/0ac2f6a0cce307d9cddc)
* [InfoQ Day 2](http://www.infoq.com/news/2014/10/devops-days-belgium-2)