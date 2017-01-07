---
title: 'AWS CodeBuild: The missing link for deployment pipelines in AWS'
date: 2016-12-19 20:50:36
---


This is a follow-up of my AWSAdvent article [Serverless everything: One-button serverless deployment pipeline for a serverless app
](https://www.awsadvent.com/2016/12/14/serverless-everything-one-button-serverless-deployment-pipeline-for-a-serverless-app/), which extends the example deployment pipeline with AWS CodeBuild.

Deployment pipelines are very common today, as they are usually part of a continuous delivery/deployment workflow. While it's possible to use e.g. projects like [Jenkins](https://jenkins.io/) or [concourse](https://concourse.ci/) for those pipelines, I prefer using managed services in order to minimize operations and maintenance so I can concentrate on generating business value. Luckily, AWS has a service called CodePipeline which makes it easy to create deployment pipelines with several stages and actions such as downloading the source code from GitHub, and executing build steps.

For the build steps, there are several options like invoking an external Jenkins Job, or SoranoCi etcpp. But when you want to stay in AWS land, your options were quite limited until recently. The only pure AWS option for CodePipeline build steps (without adding operational overhead, e.g. managing servers or containers) was invoking Lambda functions, which has several drawbacks that I all experienced:
 
## Using Lambda as Build Steps

### 5 minutes maximum execution time
Lambda functions have a limit of 5 minutes which means that the process gets killed if it exceeds the timeout. Longer tests or builds might get aborted and thus result in a failing deployment pipeline. A possible workaround would be to split the steps into smaller units, but that is not always possible. 

### Build tool usage

The NodeJS 4.3 runtime in Lambda has the `npm` command pre-installed, but it needs several hacks to be working. For example, the Lambda runtime is a read-only file system except for `tmp`, so in order to use e.g. NPM, you need to fake the `HOME` to `/tmp`. Another example is that you need to find out where the preinstalled NPM version lives ([checkout my older article on NPM in Lambda](https://www.ruempler.eu/2016/07/18/how-to-install-and-use-a-newer-version-3-x-of-npm-on-aws-lambda/)).

### Artifact handling

CodePipeline works with so called artifacts: Build steps can have several input and output artifacts each. These are stored in S3 and thus have to be either downloaded (input artifact) or uploaded (output artifact) by a build step. In a Lambda build step, this has to be done manually, means you have to use the S3 SDK of the runtime for artifact handling.

### NodeJS for synchronous code

When you want to use a preinstalled NPM in Lambda, you need to use the NodeJS 4.3 runtime. At least I did not manage to get the preinstalled NPM version running which is part of the Lambda Python runtime. So I was stuck with programming in NodeJS. And programming synchronous code in NodeJS did not feel like fun for me: I had to learn how promises work for code which would be a few lines of Python or Bash. When I look back, and there would be still no CodeBuild service, I would rather invoke a Bash or Python script from within the NodeJS runtime in order to avoid writing async code for synchronous program sequences.

### Lambda function deployment

The code for Lambda functions is usually packed as ZIP file and stored in an S3 bucket. The location of the ZIP file is then referenced in the Lambda function. This is how it looks in CloudFormation, the Infrastructure-as-Code service from AWS:

```
  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        S3Bucket: !Ref DeploymentLambdaFunctionsBucket
        S3Key: !Ref DeploymentLambdaFunctionsKey

```

That means there has to be another build and deployment procedure which packs and uploads the Lambda function code to S3 itself. Very much complexity for a build script which is usually a few lines of shell code, if you ask me.

By the way, actually there is a workaround: In CloudFormation, it's possible to specify the code of the Lambda function inline in the template, like this:

```
  LambdaFunctctionWithInlineCode:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        ZipFile: |
          exports.handler = function(event, context) {
            ...
          }
```
While this has the advantage that the pipeline and the build step code are now in one place (the CloudFormation template), this comes at the cost of losing e.g. IDE functions for the function code like syntax checking and highlighting. Another point: the inline code is limited to 4096 characters length, a limit which can be reached rather fast. Also the CloudFormation templates tend to become very long and confusing. In the end using inline code just felt awkward for me ...

### No AWS CLI installed in Lambda

Last but not least, there is no AWS CLI installed in the Lambda runtime, which makes things to be done in build steps, like uploading directories to S3, really hard, because this has to be done in the programming runtime. What would be a one-liner in AWS CLI, can be much more overhead and lines of code in e.g. NodeJS or Python.

## AWS CodeBuild to the rescue: the missing link

At the recent re:invent conference, [AWS announced CodeBuild](https://aws.amazon.com/blogs/aws/aws-codebuild-fully-managed-build-service/) which is a build service, very much like a managed version of Jenkins, but **fully integrated into the AWS ecosystem**. Here are a few highlights:

 - **Fully integrated into AWS CodePipeline:** CodePipeline is the "Deployment Pipeline" service from AWS and supports CodeBuild as an action in a deployment pipeline. It  also means that CodePipeline can checkout code from e.g. a GitHub repository first, save it as output artifact and pass it to CodeBuild, so that the entire artifact handling is managed, no (un)zipping and S3 juggling necessary.
 - **Managed build system based on Docker Containers:** First you don't need to take care of any Docker management. Second you can either use AWS provided images, which provide a range of operating systems / environments, e.g. Amazon Linux and Ubuntu for several pre-built environments, e.g. NodeJS or Python or Go ([http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref.html](list of all environments)). Or you can bring your own container (I did not try that out yet).
 - **Fully supported by CloudFormation**, the Infrastructure-as-Code service from AWS: You can codify CodeBuild projects so that they are fully automated, and reproducible without any manual and error-prone installation steps. Together with CodePipeline they form a powerful unit to express entire code pipelines as code which further reduces total cost of ownership.
 - **YAML-DSL**, which describes the build steps (as a list of shell commands), as well as the output artifacts of the build.
 
Another great point is that the provided images are very similar to the Lambda runtimes (based on Amazon Linux) so that they are predestinated for tasks like packing and testing Lambda function code (ZIP files).

### CodeBuild in action

So, what are the particular advantages of using CodeBuild vs. Lambda in CodePipeline? Have a look at [this Pull Request](https://github.com/s0enke/cloudformation-templates/pull/9/files). It replaces the former Lambda-based approach with CodeBuild in the project I set up for my AWS Advent article: Several hundred lines of JavaScript got replaced by some lines of CodeBuild YAML. Here is how a sample build file looks:

```yaml
version: 0.1
phases:
    install:
      commands:
        - npm install -g serverless
        - cd backend && npm install
    build:
      commands:
        - "cd backend && serverless deploy"
        - "cd backend && aws cloudformation describe-stacks --stack-name $(serverless info | grep service: | cut -d' ' -f2)-$(serverless info | grep stage: | cut -d' ' -f2) --query 'Stacks[0].Outputs[?OutputKey==`ServiceEndpoint`].OutputValue' --output text > ../service_endpoint.txt"
artifacts:
  files:
    - frontend/**/*
    - service_endpoint.txt
```

This example shows a `buildspec.yml` with two main sections: `phases` and `artifacts`:

 - `phases` apparently lists the phases of the build. These predefined names actually have no special meaning and you can put as many and arbitrary commands into it. The example shows several shell commands executed, in particular first - in the `install` stage - the installation of the `serverless` NPM package, followed by the `build` stage which contains the execution of the Serverless framework (`serverless deploy`). Lastly, it runs a more complex command to save the output of a CloudFormation stack into a file called `service_endpoint.txt`: That file is later picked up as an output artifact.
 - `artifacts` lists the directories and files which CodePipeline will generate as an output artifact. Used in combination with CodePipeline, it provides a seamless integration into the pipeline and you can use the artifact as input for another pipeline stage or action. In this example the `frontend` folder and the mentioned `service_endpoint.txt` file are nominated as output artifacts.
 
The `artifacts` section can also be omitted, if there are no artifacts at all.

Now that we learned the basics of the `buildspec.yml` file, lets see how this integrates with CloudFormation:

### CodeBuild and CloudFormation

CloudFormation provides a type `AWS::CodeBuild::Project` to describe CodeBuild projects - an example follows:
```yaml
  DeployBackendBuild:
    Type: AWS::CodeBuild::Project
    Properties:
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/eb-nodejs-4.4.6-amazonlinux-64:2.1.3
        Type: LINUX_CONTAINER
      Name: !Sub ${AWS::StackName}DeployBackendBuild
      ServiceRole: !Ref DeployBackendBuildRole
      Source:
        Type: CODEPIPELINE
        BuildSpec: |
          version: 0.1
          ...
```

This example creates a CodeBuild project which integrates into a CodePipeline (`Type: CODEPIPELINE`), and which uses a AWS provided image for `nodejs` runtimes. The advantage is that e.g. NPM is preinstalled. The `Source` section describes again that the source code for the build is coming from a CodePipeline. The `BuildSpec` specifies in inline build specification (e.g. the one shown above).

You could also specify that CodeBuild should search for a `buildspec.yml` in the provided source artifacts rather than providing one via the project specification.

### CodeBuild and CodePipeline

Last but not least, let's have a look at how CodePipeline and CodeBuild integrate by using an excerpt from [the CloudFormation template](https://github.com/s0enke/cloudformation-templates/blob/master/templates/pipeline-serverless-backend-npm-frontend.yml) which describes the pipeline as code:

```yaml
Pipeline:
  Type: AWS::CodePipeline::Pipeline
  Properties:
    ...
    Stages:
      - Name: Source
        Actions:
          - Name: Source
            InputArtifacts: []
            ActionTypeId:
              Category: Source
              Owner: ThirdParty
              Version: 1
              Provider: GitHub
            OutputArtifacts:
              - Name: SourceOutput
      - Name: DeployApp
        Actions:
        - Name: DeployBackend
          ActionTypeId:
              Category: Build
              Owner: AWS
              Version: 1
              Provider: CodeBuild
          OutputArtifacts:
            - Name: DeployBackendOutput
          InputArtifacts:
            - Name: SourceOutput
          Configuration:
              ProjectName: !Ref DeployBackendBuild
          RunOrder: 1

```

This code describes a pipeline with two stages: While the first stage checks out the source code from a Git repository, the second stage is the interesting one here: It describes a stage with a CodeBuild action which takes the `SourceOutput` as input artifact, which ensures that the commands specified in the build spec of the referenced `DeployBackendBuild` CodeBuild project can operate on the source. `DeployBackendBuild` is the actual sample project we looked at in the previous section.

## The Code

The [full CloudFormation template describing the pipeline is on GitHub](https://github.com/s0enke/cloudformation-templates/blob/master/templates/pipeline-serverless-backend-npm-frontend.yml). You can actually test it out by yourself by following the instructions in the [original article](https://www.awsadvent.com/2016/12/14/serverless-everything-one-button-serverless-deployment-pipeline-for-a-serverless-app/). 

## Summary 
 
Deployment pipelines are as valuable as the software itself as they ensure reliable deployments, experimentation and fast time-to-market. So why shouldn't we treat them like software, namely **as code**. With Codebuild, AWS completed the toolchain of building blocks which are necessary to codify and automate the setup of deployment pipelines for our software:
 - without the complexity of setting up / maintaining third party services
 - no error-prone manual steps
 - no management of own infrastructure like Docker clusters as "build farms".
 - no bloated Lambda functions for build steps
 
This article showcases a CloudFormation template which should help the readers to get started with the own CloudFormation/CodePipeline/CodeBuild combo which provisions within minutes. There are no excuses anymore for manual and/or undocumented software deployments within AWS anymore ;-)
