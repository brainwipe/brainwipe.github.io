---
title: "From GitHub to AWS ECS Serverless - Part 1 - Build"
date: 2024-11-12
categories: codebuild aspnet serverless aws
---

These are technical notes for going from code committed into a repository to Blue/Green deployments on AWS as of 12th November 2024. Technology does change and some of the limitations I'm going to list will be fixed by AWS in the future.

I'm a practitioner, not an expert. If you know a better way, then go for it. Some of the instructions here are oddly specific to my use case. However, if I were to document all use cases, you would end up with the AWS documentation rather than a single thought out description.

> You might incur a cost following these instructions

## Pre-Requisites

You have the following:

- ASP.NET application that runs up in a single container
- A container that performs your ASP.NET image building
- The application is stored in a private GitHub repository 
- You have GitHub administrator access over the repository
- You have AWS permissions to use the CodePipeline and CodeBuild (but likely you'll need more)

## AWS Developer Tools in Brief

AWS has a suite of tools they call [AWS Developer Tools](https://aws.amazon.com/products/developer-tools/), which allow you to host, build, test and deploy your code with all of the usual AWS trappings of AWS CloudWatch for logging, S3 for storage, IAM for access control, SNS for sending out messages and so on. Each of the tools (CodeCommit, CodeArtifact, CodeBuild, CodeDeploy, CodePipeline) work independently so you can swap out parts by integrating with 3rd parties. If you are building your containers using [GitHub Actions](https://github.com/features/actions) then pushing them into the AWS Elastic Container Registry (ECR) then that should work. I'm not doing that here for simplicity.

We use GitHub, so we don't need CodeCommit or CodeArtifact. We do want CodeBuild, CodeDeploy and CodePipeline. They aren't as tightly integrated as you might expect for example:

> If you create a CodeBuild project in CodeBuild directly, then it won't necessarily run CodePipeline. You must create the CodeBuild Project through CodePipeline.

And vice versa - a CodeBuild project that is built in the pipeline is unlikely to build on its own because there are parameters (such as the source) that CodeBuild needs but will exist only in the pipeline.

## Building ASP.NET containers

You have followed the official [Containerize .NET App](https://learn.microsoft.com/en-us/dotnet/core/docker/build-container?tabs=windows) instructions and have a docker file that builds a docker image. AWS CodeBuild is going to use this file to build the container "runtime image" and put that container into the AWS Elastic Container Registry.

```docker
# This is from the [Containerize .NET App tutorial])(https://learn.microsoft.com/en-us/dotnet/core/docker/build-container?tabs=windows).

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build-env
WORKDIR /App

# Copy everything
COPY . ./
# Restore as distinct layers
RUN dotnet restore
# Build and publish a release
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /App
COPY --from=build-env /App/out .
ENTRYPOINT ["dotnet", "DotNet.Docker.dll"]
```

### Use AWS Image Repository rather than Docker

If you are also using node to build some embedded ReactJS front end (like we do), you might have a line like this:

`FROM node:21-alpine3.19 AS build-node`

This will pull a docker image from the docker hub by default (the Microsoft ones come from mcr.microsoft.com). You are likely to get an error message `CannotPullContainerError: You have reached your pull rate limit` [details here](https://repost.aws/knowledge-center/ecs-pull-container-error-rate-limit). To get around the rate limit, find the same container on the [AWS Public ECR gallery](https://gallery.ecr.aws/).

So it became:

`FROM public.ecr.aws/docker/library/node:21-alpine AS build-node`

## Connect AWS to GitHub

You need to authenticate AWS to GitHub once for all your pipelines. It's called a Connection and can be found in Developer Tools -> Settings -> [Connections](https://eu-west-2.console.aws.amazon.com/codesuite/settings/connection). To create a GitHub connection, you will need to be signed in with the account that has owner/administrator access of the GitHub organisation.

> You use the administrator role to setup the connection, after that AWS is installed as a GitHub App in the GitHub organisation.

So don't worry about the administrator account changing in the future.

### Steps

- Go to the [Connections Screen](https://console.aws.amazon.com/codesuite/settings/connection)
- Click Create Connection
- Choose GitHub
- Give you connection name, I recommend giving it the name of your application rather than just calling it GitHub
- Click Connect to GitHub
- Click Install a New App
- Click through the GitHub steps for installing an app onto an account (either your personal or the organisation)

At the end, you should have your new [GitHub Connection](https://console.aws.amazon.com/codesuite/settings/connection) listed on the connections screen.

## Pipelines

A pipeline is a list of steps where the result of each step feeds into the next. You can have lots of pipelines and they can run simulataneously. Before you start clicking, you need to know what you want to achieve from your pipelines. Some examples include:

- Build and deploy automatically
- Build but only deploy when signed off
- Build only
- Deploy only
- Deploy to specific environment (e.g. dev, test, staging, demo or production)

For this example, we want a build pipeline and a separate deploy pipeline.

## CodePipeline for CodeBuild

We're going to create the CodeBuild project via the CodePipeline so that the source is set correctly.

- Go to the [Pipelines screen](https://console.aws.amazon.com/codesuite/codepipeline/pipelines) and click Create Pipeline
- Select Build Custom Pipeline, Next
- For a pipeline name, describe what actions the pipeline is going to do and the product you are working on such as `deploy-myapp`
- Leave the rest of the defaults, click Next
- For Source Provider, choose GitHub App
- Select the connection (there's probably only one if you're new to this)
- Choose the Repository where you ASP.NET app is
- Choose Full clone, which gives us more features later
- Choose your trigger settings (see note below)
  - Specify filter
  - Push
  - Tags
  - Include `release*`
- Next
- Select `Other Build Providers` and choose `AWS CodeBuild` from the dropdown
- Hit Create Project, and this will take you over to CodeBuild in a popup

### A note on Service Roles

If you have a lot of pipelines, it is worth sharing service roles as they will largely be identical. Where the service roles will differ is in S3 access to where the artifacts are stored. I found that creating multiple service roles and then merging them together in the AWS IAM console on a per-product basis was easier. There's no harm in having lots of service roles (one per pipeline).

### Choosing a Trigger

Specifying a trigger will depend on how you use git to build your software. My preference is a [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow). When we want a build to go to test (QA) and beyond, we give it a tag, usually including the date and an ordinal number such as `release-2024.11.11.0`. Therefore, for the trigger defined above, we want to create a container build whenever it detects a tag with `release` in it. It uses `*` as a wildcard for matching.

### Add more containers later

If you GitHub repository contains more than one container then you can add more containers later once you get to the end of the Pipeline wizard.

## CodeBuild Project

We're now going to create the CodeBuild project, which will use your `dockerfile` to build a single container and put it into the AWS Elastic Container Registry (ECR).

- For the `project name`, use `build-myapp` because it will be easier to differentiate it
- Select Managed Image (see below)
- Create a new Service Role (see below for permissions)
- Use a buildspec file - the individual build commands 
- Put in the path to your buildspec file, if the repo has multiple containers in it then you will need one buildspec for each container
- Click continue to CodePipeline to go back
- Click Add Environment variables to add any environment variables (see below)
- Click Next, Skip deploy stage (we're not doing deploy yet)
- Click Review and then Create will both create the pipeline and run it

### Managed Docker Image

This bit is a little confusing, so stick with me. This [AWS Managed Image](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html) is the docker container that will host and run your dockerfile. That means Amazon starts with an Amazon Linux Docker Container that will run up your docker file that will in turn build a "build-env" which then builds your "runtime" container. That runtime container is then put into ECR but we'll come onto that next. 

### Code Build Service Role

While AWS will create a service role for you, it won't necessarily have all the permissions it needs. Once you have build the pipeline, go to IAM and find the CodeBuild service role that you made in this step. 

This policy allows CodeBuild to read your code from GitHub via the Code Connections application. Add a policy with the following:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "codestar-connections:UseConnection",
            "Resource": "insert code connection ARN here"
        }
    ]
}
```

You might have spotted the reference to AWS CodeStar. AWS CodeStar is an old name for AWS CodeConnections, [which was renamed in March 2024](https://docs.aws.amazon.com/dtconsole/latest/userguide/rename.html).

Secondly, you must add a policy so that CodeBuild can push the final container to the AWS Elastic Container Registry. I am using `AmazonEC2ContainerRegistryFullAccess` and then I use the IAM Analyse functionality to create custom policy with only the permissions that the service role needs.

The CodeBuild error `CLIENT_ERROR: authorization failed for primary source and source version` is what you get when you miss this step.

### Buildspec File

The `buildspec` file does the heavy lifting of the build process. It's split into three sections, we'll look at each below. Don't forget that these commands are running inside a docker container shell and because we've chosen the default Amazon Linux shell, it has pre-loaded utilities on it. If you want to use a custom one, chances are you'll need to install those things using linux-like commands.

The `buildspec` file might look like a dockerfile, but it's not! It can do a lot more than listed here, we're just trying to get our container to build and push to the ECR. [Checkout the Buildspec reference for more](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html).


```yml
version: 0.2

phases:
  pre_build:
    on-failure: ABORT
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    on-failure: ABORT
    commands:
      - echo Build started on `date`
      - echo Building the Docker image for $TagName...          
      - docker build -t $IMAGE_REPO_NAME:$TagName -f myapp.dockerfile .
      - docker tag $IMAGE_REPO_NAME:$TagName $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$TagName      
  post_build:
    on-failure: ABORT
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$TagName
```

#### Passing variables into the buildspec

Variables are passed in from outside using the `$`. There are a number of built in [environment variables](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html) CodeBuild will put in there. 
If you want to know what all the variables there are, copy and paste the list below into the `pre_build: commands:` and it will print them all out when you run the build for the first time.

```yml
- echo CODEBUILD_BATCH_BUILD_IDENTIFIER $CODEBUILD_BATCH_BUILD_IDENTIFIER
- echo CODEBUILD_BUILD_ARN $CODEBUILD_BUILD_ARN
- echo CODEBUILD_BUILD_ID $CODEBUILD_BUILD_ID
- echo CODEBUILD_BUILD_IMAGE $CODEBUILD_BUILD_IMAGE
- echo CODEBUILD_BUILD_NUMBER $CODEBUILD_BUILD_NUMBER
- echo CODEBUILD_BUILD_SUCCEEDING $CODEBUILD_BUILD_SUCCEEDING
- echo CODEBUILD_INITIATOR $CODEBUILD_INITIATOR
- echo CODEBUILD_KMS_KEY_ID $CODEBUILD_KMS_KEY_ID
- echo CODEBUILD_LOG_PATH $CODEBUILD_LOG_PATH
- echo CODEBUILD_PUBLIC_BUILD_URL $CODEBUILD_PUBLIC_BUILD_URL
- echo CODEBUILD_RESOLVED_SOURCE_VERSION $CODEBUILD_RESOLVED_SOURCE_VERSION
- echo CODEBUILD_SOURCE_REPO_URL $CODEBUILD_SOURCE_REPO_URL
- echo CODEBUILD_SOURCE_VERSION $CODEBUILD_SOURCE_VERSION
- echo CODEBUILD_SRC_DIR $CODEBUILD_SRC_DIR
- echo CODEBUILD_START_TIME $CODEBUILD_START_TIME
- echo CODEBUILD_WEBHOOK_ACTOR_ACCOUNT_ID $CODEBUILD_WEBHOOK_ACTOR_ACCOUNT_ID
- echo CODEBUILD_WEBHOOK_BASE_REF $CODEBUILD_WEBHOOK_BASE_REF
- echo CODEBUILD_WEBHOOK_EVENT $CODEBUILD_WEBHOOK_EVENT
- echo CODEBUILD_WEBHOOK_MERGE_COMMIT $CODEBUILD_WEBHOOK_MERGE_COMMIT
- echo CODEBUILD_WEBHOOK_PREV_COMMIT $CODEBUILD_WEBHOOK_PREV_COMMIT
- echo CODEBUILD_WEBHOOK_HEAD_REF $CODEBUILD_WEBHOOK_HEAD_REF
- echo CODEBUILD_WEBHOOK_TRIGGER $CODEBUILD_WEBHOOK_TRIGGER
- echo CODEBUILD_RUNNER_OWNER $CODEBUILD_RUNNER_OWNER
- echo CODEBUILD_RUNNER_REPO $CODEBUILD_RUNNER_REPO
- echo CODEBUILD_RUNNER_REPO_DOMAIN $CODEBUILD_RUNNER_REPO_DOMAIN
- echo CODEBUILD_WEBHOOK_LABEL $CODEBUILD_WEBHOOK_LABEL
- echo CODEBUILD_WEBHOOK_RUN_ID $CODEBUILD_WEBHOOK_RUN_ID
- echo CODEBUILD_WEBHOOK_JOB_ID $CODEBUILD_WEBHOOK_JOB_ID
- echo CODEBUILD_WEBHOOK_WORKFLOW_NAME $CODEBUILD_WEBHOOK_WORKFLOW_NAME
- echo CODEBUILD_RUNNER_WITH_BUILDSPEC $CODEBUILD_RUNNER_WITH_BUILDSPEC
```

You can pass your own variables into your buildspec, you'll notice that . If you make your own, they cannot begin with `CODEBUILD_` or `AWS_` as they are reserved. The environment variables are not specified in the CodeBuild popup when using CodePipeline (they would be if you were only using CodeBuild), they are specified in the parent window. Do those later!

I needed to add the following environment variables into the build:

- `IMAGE_REPO_NAME` the ECR image name
- `TagName`
- `AWS_ACCOUNT_ID`
- `AWS_DEFAULT_REGION`

They would not be pre-filled by AWS when running CodeBuild through CodePipeline.

#### Pre-Build log-in to AWS ECR

The default CodeBuild container will have AWS toolkit installed on it but it does not automatically have acccess to the ECR, so you need to login. The container runs under the service role you created in the previous step and that will need to have access to the ECR (it did by default when I set it up). It uses that service role to log into the ECR. Strange, I know, but it's how this works.

#### Adding Source Variable Environment Variables

We have seen how the `buildspec` file needs `$TagName` environment variable. This is the git tag that triggered this whole process in the first place. To set the tagname correctly, you must create an environment variable in the CodePipeline build window, using `TagName` for the name (ignore the $) and then for the value use:

```
#{SourceVariables.TagName}
```

The `#{}` tells AWS that it needs to do a replacement. `SourceVariables` is the name of the "Variable Namespace" given to the variables from the Source. [Source Variables reference here](https://docs.aws.amazon.com/codepipeline/latest/userguide/reference-variables.html). You can't see them while you're building the pipeline but once the source part of the pipeline has run, they will appear. The reference doesn't include all the variables I am getting from the GitHub App source. Here's a list of what I see:

```
- AuthorDate          2024-11-11T14:48:38Z
- AuthorDisplayName   Rob Lang
- AuthorEmail         brainwiped@gmail.com
- AuthorId            Rob Lang
- BranchName          feature/blog
- CommitId            91d21ea05d4bef9c7c80d90f96cb0cacf9c9c90d
- CommitMessage       Blogging for the CodePipeline build
- ConnectionArn       arn:aws:codeconnections:eu-north-1:123123123123:connection/7b65fd00-061e-43c7-8d42-e55dd18843a3
- FullRepositoryName  brainwipe/blog
- ProviderType        GitHub
- TagName             release-20241111
```

So I could include any of these Source Variables in my buildspec by adding an environment variable such as `#{SourceVariables.TagName}` or `#{SourceVariables.CommitId}`. This could be useful if you want to tag your ECR image with both the tag and the commit id.

> Warning: when using TagName, it will only appear on the image if you push to git. If you then request a future build through the AWS Console, the git tag will not be there.

## Final Thoughts

If you build the CodeBuild project through CodePipeline then you can't run that project inside CodeBuild alone. That is because it won't have the source information or the environment variables as they are held in the CodePipeline. While that feels odd, I did not actually need to use the CodeBuild build button once I had the pipeline up and running.

I appreciate that this is an information dump, I hope it is useful on your journey and that by the time you're reading this, AWS has adjusted its UI and build process so that it's easier to manage.