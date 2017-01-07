---
title:  Keeping your Pocket list clean with pocketcleaner and AWS Lambda
date: 2016-01-10
---
Over the last years my Pocket reading queue got longer and longer. It actually dated back to stuff from 2013\. Over the time a realized I would never ever be able to keep up with it again.

Some days ago I found out that Daniel (mrtazz) developed a nice tool named [pocketcleaner](https://github.com/mrtazz/pocketcleaner "https://github.com/mrtazz/pocketcleaner") which archives too old Pocket entries. I thought "Hey great, that's one solution to my problem, but how to execute it?". People who know me might already have an idea :) I don't like servers in terms infrastructure that I have to maintain. So I thought **AWS Lambda to the rescue**!

And here it is: An [Ansible playbook](https://github.com/s0enke/pocketcleaner-ansible "https://github.com/s0enke/pocketcleaner-ansible") which setups a Lambda function which downloads, configures and executes the Go binary. It can be triggered by a AWS event timer. No servers, just a few cents per month (maximum!) for AWS traffic and Lambda execution costs.
