---
title:  New AWS CloudFormation YAML syntax and variable substitution in action
date: 2016-09-26
---
I've been using CloudFormation YAML syntax for a while now with [Ansible](http://docs.ansible.com/ansible/cloudformation_module.html "http://docs.ansible.com/ansible/cloudformation_module.html") and the [serverless framework](https://serverless.com/ "https://serverless.com/") which would convert the YAML to JSON before uploading the template. That already gave me the YAML advantages of e.g. code comments, not having to care about commas etc.

A few days ago, AWS announced native [YAML support for CloudFormation templates](https://aws.amazon.com/blogs/aws/aws-cloudformation-update-yaml-cross-stack-references-simplified-substitution/ "https://aws.amazon.com/blogs/aws/aws-cloudformation-update-yaml-cross-stack-references-simplified-substitution/"), in addition to the existing JSON format.

And along with that they added **new shorthand syntax for several functions**. 

Let's go through [**a template**](https://github.com/s0enke/serverless-freifunk-alarm/blob/master/app.yaml "https://github.com/s0enke/serverless-freifunk-alarm/blob/master/app.yaml") which I created not only in order to get used to the new syntax :)

## Injecting "arguments" to inline Lambda functions

One of the real powers of Lambda and CloudFormation is that you can use Lambda to add almost any missing functionality to CloudFormation (e.g. [custom resources](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources-lambda.html "http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources-lambda.html")), or to create small functions, without having to maintain another deployment workflow for the function (In this example I created an Lambda function which polls some web services and writes the result into a CloudWatch custom metric.)

The interesting part is how `AccessPointName` is injected into the Lambda function (in this example some Python code). We are making use of the [new short substitution syntax](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html "https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html") here which allows us to replace CloudFormation references with their value:

    
     CheckProgram:
       Type: AWS::Lambda::Function
       Properties:
         Code:
           ZipFile: !Sub |
             ...
             def handler(event, context):  
               ...
               found_access_points = [access_point for access_point in api_result["allTheRouters"] if access_point["name"] == "${AccessPointName}"]
    

In this example the variable "`AccessPointName`" gets then substituted by the value (in this particular case a stack parameter). Please also mind the "|" which is no special CloudFormation syntax but **multi line YAML syntax**.

[Throughout the
template](https://github.com/s0enke/serverless-freifunk-alarm/blob/master/app.yaml "https://github.com/s0enke/serverless-freifunk-alarm/blob/master/app.yaml") you can find other usage examples of the new substitution syntax, for example a cron job with CloudWatch events which gets:

    
     CheckProgramTrigger:
       Type: AWS::Events::Rule
       Properties:
         ScheduleExpression: !Sub rate(${CheckRateMinutes} minutes)
         Targets:
           - Arn:
               !GetAtt [CheckProgram, Arn]
             Id: InvokeLambda
    

## Referencing with the !Ref and !GetAttr shortcuts

Another feature addition is a short hand syntax for `Ref` and `GetAttr` calls.

    
     AccessPointOfflineAlertTopic:
       Type: AWS::SNS::Topic
       Properties:
         Subscription:
           - Endpoint: !Ref NotificationEmail
             Protocol: email
    

This example creates an SNS topic with an email subscription which is once again a CloudFormation template parameter.

## Recap

With the new syntax it's now possible to create YAML syntax, and we have nice shortcuts for commonly used functions. My personal highlight is the shorthand substitution syntax, esp. when using inline Lambda functions.
