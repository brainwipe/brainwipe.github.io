---
title: "From GitHub to AWS ECS Serverless - Part 2 - Deploy"
date: 2024-12-11
categories: codebuild aspnet serverless aws codedeploy codepipeline
---

These are technical notes for going from code committed into a repository to Blue/Green deployments on AWS as of 20th December 2024. Technology does change and some of the limitations I'm going to list will be fixed by AWS in the future.

I'm a practitioner, not an expert. If you know a better way, then go for it. Some of the instructions here are oddly specific to my use case. However, if I were to document all use cases, you would end up with the AWS documentation rather than a single thought out description.

> You might incur a cost following these instructions

[Part 1 covering build here](/2024-11-24-build-aspnet-codebuild)

## Pre-Requisites

- You have a build pipeline that takes your code, builds a new container and puts it into the AWS Elastic Container Registry (ECR)
- You already have a service running in AWS Elastic Container Service (ECS)
- It doesn't matter how the service got there - either through the CLI, Console or by Cloud Formation.


## There are 2 different ways to deploy to ECS

1. Amazon ECS
2. Amazon ECS Blue/Green

While the names are the same, how they deploy is very very different. Amazon ECS goes straight to ECS. Amazon ECS Blue/Green uses AWS Code Deploy. The AWS document does not make this clear.

Each deployment is a step in the pipe and you can run both together - deploying a service and an API together but they work very differently!

### Amazon ECS overview
An Amazon ECS standard deployment is for when you want to replace a running service or run a one-shot task without the need of complex blue/green updates. 

For Amazon ECS, you need to have an image [definitions file](https://docs.aws.amazon.com/codepipeline/latest/userguide/file-reference.html). This file maps the image URI in the Elastic Container Registry and container name to the task definition. It's usually called `imagedefinitions.json`.

[Official tutorial for the standard ECS deployment](https://docs.aws.amazon.com/codepipeline/latest/userguide/ecs-cd-pipeline.html).

### Amazon ECS Blue/Green overview
An Amazon ECS Blue/Green deployment is when you have services behind a load balancer (such as an API) and you want to gradually swap out the existing (blue) with a new deployment (green).

For Amazon ECS Blue/Green, you use AWS Code Deploy, which needs an "AppSpec" file, which specifies the name of the container, task definition etc.

[Official tutorial for ECS Blue/Green deployment](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-ecs-ecr-codedeploy.html)

## Pipeline using Amazon ECS

These are the steps and decisions you have to make to deploy a .NET service on a container in ECS using the standard ECS deployment.

Pre-requisites:

- A container stored in the ECR with a .NET service on it, we'll call this the worker
- The container has a tag that doesn't change with a build
- A GitHub repository to put the `imagedefinitions.json` file in

> Your pipeline will fail at first because you don't have imagedefinitions file, don't worry, we add it after.

### Steps

This only builds part of the pipeline, we'll add more parts after.

- Create a new pipeline
- Set the pipeline name to a description of what it will be doing, such as deploying a worker.
- Leave all the defaults and click Next
- Source provider choose Amazon ECR
- Choose the repository with the container
- For the image tag, set a tag that you know won't change. Don't leave it blank!
- Skip build stage (we've already done that)
- Deploy Provider choose Amazon ECS
- Add your ECS cluster name and the ECS service name
- Leave Image Definitions file blank for now!
- Next/Create

### Getting the image definitions file

You need to put the image definitions file somewhere where it can be loaded by the pipeline. If you were building and deploying in one go then a `buildspec.yml` from the build could output one. However, we don't have that luxury because the build happened some time before.

We keep our image definition in its own GitHub repository (we call infrastructure). We keep it away from the main codebase because the main codebase is concerned only with creating the containers, not the infrastructure used to host or deploy them. You can also keep it on S3 but it's worth controlling it as code.

The image definition file can have multiple containers in it but in this example, we're using only one. It looks like:

```json
[{"name":"worker","imageUri":"1234567890.dkr.ecr.eu-west-2.amazonaws.com/mco/worker:test"}]
```

Where `name` is the name of the service in ECS we're deploying to and `imageUri` is ECR image locaiton. You'll notice we're using the tag "test" and that won't change. See "You can't specify ECR tag as a Pipeline parameter" below for why.

With that saved in GitHub, go back to the pipeline and hit edit.

- In Source, click Edit stage
- Click `+ Add Action Group`
- Action name `Infrastructure GitHub`
- Action Provider GitHub (via GitHub app)
- Connection as setup, with your new infrastructure GitHub repo and branch.
- Select CodePipeline Default (we're not building here)
- Variable namespace it's worth calling it `InfrastructureVariables`
- Output artifacts called `InfrastructureArtifacts`
- Click done

For Variables and Artifacts names are so that you can tell the different sources apart in the deployment step.

### Giving deploy the image definitions file

So far the pipeline has the image definitions file but it is not using it.

- In Deploy, click Edit stage
- Click Edit on the deploy action (the little pencil icon)
- For input artifacts, select infrastructure
- Under image definitions file, put the path to the `imagedefinitions.json` in the GitHub repository with respect to the root.
- Click done.

You're now ready to deploy the service! It will always deploy the ECR image with the tag "test".

## Pipeline using Amazon ECS Blue/Green

The Blue/Green deployment is different because it needs to configure the load balancer's deployment groups, making sure that the new (green) containers are linked properly and the old (blue) containers are switched off and removed.

Pre-requisites:

- A container stored in the ECR with a .NET service on it, we'll call this the API
- The container has a tag that doesn't change with a build
- A GitHub repository to put the `imagedefinitions.json` file in
- Load balancer with 2 groups

### Steps

The wizard is going to create a lot of stuff for you!

> The deployment will fail on first run!

- Create a new pipeline
- Set the pipeline name to a description of what it will be doing, such as deploying an API.
- Leave all the defaults and click Next
- Source provider choose Amazon ECR
- Choose the repository with the container
- For the image tag, set a tag that you know won't change. Don't leave it blank!
- Skip build stage (we've already done that)
- Deploy Provider choose Amazon ECS Blue/Green

It's now going to create an AWS Code Deploy application (if you don't already have one)

- Hit Create Application 
- Name it API and choose Amazon ECS
- Click create

You're now in AWS Code Deploy and you must create a deployment group (there might be a green message at the top telling you to)

- Click create deployment group
- Group name as API
- Choose a service role that can access your containers
- Choose cluster name
- Choose the service name
- Choose your load balance and the deployment groups (you should already have these)
- Create deployment group

Leave Task Definition and AppSpec for now (do those below)

- Set Input artifact to the SourceArtifacts
- Set placeholder to IMAGE_NAME
- Click create

There are two files we need to sort out now, the Task Definition and the App Spec file.

### ECS Task Definition file

The task definition file is a description of the resources the running container in ECS is going to need. When you setup your ECS service, you probably created one (or had one created by a wizard for you). It's over in ECS under Task Definitions. 

Every time there is an ECS deployment (either standard or Blue/Green) a new revision of the task definition is made, even if there haven't been any changes. Once your have the automated deployment set up, you will not be editing this task definition directly (unless in emergency) but instead updating it with a deployment. Any changes you make directly to the task definition in ECS will be overwritten by the next deployment, so make sure you move any changes back.

I recommend storing the task definitions in a new GitHub repository called Infrastructure (like we did for Amazon ECS Standard deployment).

- Edit your pipeline
- In the Source stage, click Edit Stage
- Add a new action
- Click `+ Add Action Group`
- Action name `Infrastructure GitHub`
- Action Provider GitHub (via GitHub app)
- Connection as setup, with your new infrastructure GitHub repo and branch.
- Select CodePipeline Default (we're not building here)
- Variable namespace it's worth calling it `InfrastructureVariables`
- Output artifacts called `InfrastructureArtifacts`
- Edit the Deploy definition and set Amazon ECS task definition to your taskdef.
- Click done

#### ECR container URI IMAGE_NAME replacement

You can get code pipeline to dynamically chance the ECR image name in the task definition. This is important if you are deploying a specific ECR version. In our case, we are tagging all ECR images with the "test" tag, so it does not *need* to dynamically change.

In the Task Definition, there will be a place to define the ECR image that will be deployed. The placeholder `<IMAGE_NAME>` will be replaced by the Code Pipeline action.

```json
{
  "family": "mco-api",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "<IMAGE_NAME>",
      "cpu": 0,
      "portMappings": [
  // snipped for brevity
```

- Edit your pipeline
- In the deploy stage, edit the Blue/Green deployment
- Under `Dynamically update task definition image`, choose the SourceArtifact
- Set the placeholder text to `IMAGE_NAME` (remove the angle brackets)
- DOne

### Code Deploy AppSpec

The [AppSpec file](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html) is used by Code Deploy to determine the task definition, the container and port and some optional information about your service. It's not very large, ours is:

```json
{
  "version": 0.0,
  "Resources": [
    {
      "TargetService": {
        "Type": "AWS::ECS::Service",
        "Properties": {
          "TaskDefinition": "<TASK_DEFINITION>",
          "LoadBalancerInfo": {
            "ContainerName": "api",
            "ContainerPort": 8080
          }
        }
      }
    }
  ]
}
```

The task definition is included as `<TASK_DEFINITION>` because it will be replaced by the specific versioned URN of the task definition by Code Deploy.

- Edit your pipeline
- Edit the Deploy definition and set AppSpec to your wherever you put your appspec.
- Click done

## Things you can't do

There were some annoyances that at time of writing means certain workflows are really complex to service with AWS CodePipeline.

### You can't specify ECR tag as a Pipeline parameter

If your workflow is the following:

- Tag a branch in github
- Have a pipeline that is triggered on a tag, builds the code into a container image and puts it into ECR
- Later on, I want to release that tag

You can't do that with code pipeline.

AWS Pipelines does have variables that you can pass into the pipeline but the pipeline action that pulls the image from the ECR is hardcoded to a tag. You cannot pass in the pipeline variable into the ECR action.

What this means is that a pipeline is fixed to an image tag, so you must get into the habit of overwriting tags. Avoid using "Latest" tag, that is not best practise.

The way we solved this was to tag the image twice. First, with the commit ID and then again with the destination, such as "test". Then, when you want to promote that image, you can retag it using the commit id to the next environment, such as "staging". Elastic Container Registry [tells you how to retag an image](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-retag.html). 

The downside is that the container doesn't get untagged if the deployment failed, so you can't use ECR to tell you what version actually got deployed, only the one that last tried. There are other ways to find that out, though.