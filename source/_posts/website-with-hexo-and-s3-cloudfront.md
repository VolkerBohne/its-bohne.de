---
title: Website now powered by Hexo, AWS CloudFront and S3 
date: 2016-12-18 21:23:23
tags:
---

Over the past days I moved my blog over to AWS CloudFront and S3, powered by the static blog generator [Hexo](https://hexo.io/).

Here are a few highlights:

 - The source code of the website is now open source on [GitHub](https://github.com/s0enke/ruempler.eu).
 - The infrastructure for the website is automated and codified by a [CloudFormation template](https://github.com/s0enke/cloudformation-templates/blob/master/templates/hexo-website-cdn-pipeline.yml).
 - The website is secured via HTTPS thanks to CloudFront and the Amazon Certificate Manager
 - The build of the website if entirely codified and automated with AWS CodePipeline and CodeBuild (see the CloudFormation template for details).
 - The website and building infrastructure are serverless. No servers, VMs or containers to manage.
 - Major performance enhancements since the website is now static and powered by a CDN.