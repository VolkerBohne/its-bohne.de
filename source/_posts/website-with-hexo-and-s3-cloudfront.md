---
title: Website now powered by Hexo, AWS CloudFront and S3 
date: 2018-07-22 20:08:09
tags:
---

This blog website is powered by AWS CloudFront and S3, powered by the static blog generator [Hexo](https://hexo.io/).

Here are a few highlights:

 - The source code of the website is on GitHub.
 - The infrastructure for the website is automated and codified by a CloudFormation template.
 - The website is secured via HTTPS thanks to CloudFront and the Amazon Certificate Manager
 - The build of the website is entirely codified and automated with AWS CodePipeline and CodeBuild.
 - The website and building infrastructure are serverless. No servers, VMs or containers to manage.
 - Major performance enhancements since the website is static and powered by a CDN.