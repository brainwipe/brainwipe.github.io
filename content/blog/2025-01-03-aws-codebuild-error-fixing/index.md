---
title: "AWS CodeBuild Error Fixing"
date: 2025-01-03
categories: codebuild aspnet serverless aws
---

I was expecting this to be a longer post about the issues I had but I moved onto a new task halfway through. When I return to it, I'll add more here.

## Authorisation failed for primary source

When building a contain in CodeBuild you receive an error:

```
[Container] 2025/01/03 14:10:46.285161 Waiting for DOWNLOAD_SOURCE
CLIENT_ERROR: authorization failed for primary source and source version d7e3388d....
```

This is telling you that code build does not have access to your code repository. If you're using GitHub then that means the policy attached to the code build role needs to have access to the git code connector (called Codestar for legacy reasons).

### How to fix

- Copy the ARN of your source control connection [from here](https://eu-west-2.console.aws.amazon.com/codesuite/settings/connections?connections)
    - If that link doesn't work, it's in AWS Developer Tools -> Settings -> Connections
- In CodeBuild, find your build project
- Go to Build Details tab
- Under Environment, click Service Role
- Edit the policy called CloudBuildeBasePolicy-name-of-project-awszone
- At the bottom add the following (replace URN of connection)

```json
    {
        "Effect": "Allow",
        "Action": "codestar-connections:UseConnection",
        "Resource": "[PUT ARN OF YOUR CODE CONNECTION HERE]"
    }
```

- Save policy
- Re-run the build pipeline

## ECS Task - cannot pull container error

If you are deploying a task or service to ECS and you see this:

```
CannotPullContainerError: pull image manifest has been retried 1 time(s): failed to resolve ref docker.io/mycontainer:latest: pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
```

This means that ECS is trying to pull from Docker Hub (`docker.io`), whereas your container is probably in the Elastic Container registry. You need to put the full Elastic Container Registry (ECR) URI into the task definition or it will assume you're pulling from `docker.io`.

### How to fix

- Open Elastic Container registry and find the container image you want to run a task for
- Copy the URI. It looks something like: `0123456789.dkr.ecr.eu-west-2.amazonaws.com/mycontainer`
- Open the Task Definition for the ECS task that you're running
- The top part of the file looks like:

```json
{
    "family": "mycontainer",
    "containerDefinitions": [
        {
            "name": "cli",
            "image": "mycontainer",
            "cpu": 0,
            "portMappings": [],
            "essential": true,
            "environment": [
```

- Replace image with the ECR URI so it looks more like:

```json
{
    "family": "mycontainer",
    "containerDefinitions": [
        {
            "name": "cli",
            "image": "0123456789.dkr.ecr.eu-west-2.amazonaws.com/mycontainer",
            "cpu": 0,
            "portMappings": [],
            "essential": true,
            "environment": [
```

- Save the task definition and create a new task.