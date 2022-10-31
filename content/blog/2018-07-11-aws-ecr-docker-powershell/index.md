---
title: "AWS Elastic Container Registry with docker compose on Powershell"
date: 2018-07-11
categories: docker aws
---
Amazon Web Services (AWS) [Elastic Container Registry](https://aws.amazon.com/ecr/) (ECR) is a store for your [docker images](https://docs.docker.com/glossary/?term=image). Once you create your images locally, you can push them up to the ECR and in turn make them live. In this article, we're going to use an AWS IAM account that has Multi Factor Authentication (MFA) switched on to push multiple containers to the registry using docker compose. 

## Revising Terminology
A docker _container_ runs your live application. A docker _image_ is the template that is used to create the container. It is a collection of files: executables, frameworks and static content. A docker _repository_ is a set of docker images. You distinguish between different images in the repository using _tags_. A _registry_ is a hosted service that holds repositories and their images. [Docker Hub](https://hub.docker.com/) is the default registry, we're going to use the AWS [Elastic Container Registry](https://aws.amazon.com/ecr/). _Docker compose_ is a tool for dealing with multiple images at the same time.

More detail in [the offical Glossary](https://docs.docker.com/glossary/).

## Pre-requisites
You will need the following:

- Up-to-date Powershell
- [AWS Tools for Powershell](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-getting-set-up-windows.html)
- docker for Windows! Check out [my guide](https://brainwipe.github.io/2017-10-30-oauth-on-docker-part1) for doing that.
- [MFA set up](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable.html) on you IAM account and a device that can perform 2 factor authentication (I use my phone with [Google Authenticator](https://support.google.com/accounts/answer/1066447?co=GENIE.Platform%3DAndroid&hl=en))
- The serial number of your MFA device. You can find it on the IAM user details and will look something like this: `arn:aws:iam::123123123123:mfa/Rob`

Ensure that you have created an AWS IAM user to communicate with AWS. Do not use your root user account. In the account detail for the IAM user, [create an access key and secret pair](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html). The secret will only be displayed once, so I'd copy it somewhere you know you can destroy it or leave the browser window open. 

You will then need to configure the AWS Powershell tools to use these credentials. Make sure that you're using a profile that suits your machine, I used:

    Set-AWSCredential -AccessKey THISISTHEACCESSKRY -SecretKey IMNOTPUTTINGTHESECRETONTHEINTERNET -StoreAs default

## My example App
For my example, I'm going to be using docker-compose that [I introduced in my OAuth blog post](https://brainwipe.github.io/2017-10-30-oauth-on-docker-part1). The docker-compose file creates two images. It currently looks like this: 

```yml
version: '3'

services:
dockerdotnetoauth:
    image: dockerdotnetoauth
    build:
        context: ./DockerDotNetOAuth
        dockerfile: Dockerfile

identityserver:
    image: identityserver
    build:
        context: ./IdentityServer
        dockerfile: Dockerfile
```

## Create ECR Repositories
Log into AWS and go to Elastic Container Service (ECS), you'll find ECR is listed at the end. Click **Create Repository** and give it the same name as your images (not required but saves a lot of confusion). For each image, you will get a Repository URI it will look something like:

    123123123123.dkr.ecr.eu-west-1.amazonaws.com/dockerdotnetoauth

The format is:

    <your account number>.dkr.ecr.<region>.amazonaws.com/<name of image>

These are empty repositories, they are just placeholders that you can put your images into.

## Getting images into the repositories
There are two ways to do this: tag your image or update your docker compose. The [official documentation](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html) goes into great length about tagging and pushing an existing image. Instead, we're going to update the docker compose file so that we can push more than one image at a time.

Take the repository URI you got in the previous step and use it for the image name.

```yml
version: '3'

services:
dockerdotnetoauth:
    image: 123123123123.dkr.ecr.eu-west-1.amazonaws.com/dockerdotnetoauth
    build:
        context: ./DockerDotNetOAuth
        dockerfile: Dockerfile

identityserver:
    image: 123123123123.dkr.ecr.eu-west-1.amazonaws.com/identityserver
    build:
        context: ./IdentityServer
        dockerfile: Dockerfile
```

When you next build the docker compose (as with Visual Studio 2017) it will create the images with the full name. It is a big, ugly name but there's currently no way to give it a domain CNAME. If you're concerned that your AWS account number is going to be stored in source control, then I recommend not checking it in and having it part of your dev environment setup.

## Log docker into AWS
To be able to push your images up to the registry, docker needs access to your AWS account. As we are using Multi Factor Authentication (MFA), you will need to have your device (phone) handy to get the MFA access code.

The steps the script takes are:
- Open Powershell!
- Log into AWS Secure Token Server to get an access token.
- Get the docker login command from ECR
- Invoke the docker login command

When requesting a token from AWS, you need to specify how long (in seconds) you want the token to last for. The maximum is 36 hours but it depends on your use case. If you are logging in purely to upload the latest image build then 300 seconds should be enough. If you are dev ops and uploading for a working day, then 28800 seconds (8 hours) is enough.

To get the token and store it in a Powershell variable use:

    $token = Get-STSSessionToken -DurationInSeconds <how long in seconds> -SerialNumber <mfa serial> -TokenCode <code from the authenticator on your phone>

For example:

    $token = Get-STSSessionToken -DurationInSeconds 28800 -SerialNumber arn:aws:iam::123123123123:mfa/Rob -TokenCode 123123

We're then going to get the docker login command from ECR and invoke it immediately:

     Invoke-Expression –Command (Get-ECRLoginCommand -Credential $token).Command

Your local docker is now logged into ECR.

## Push all your images via docker compose
The benefit of setting the image name in docker compose to the full ECR URI is that you can then push all of your built containers in one go. That's useful if you're using docker compose inside Visual Studio to build .NET Core Linux containers. Visual Studio doesn't care what the image names are.

To push all of your built images with the `:latest` tag up to ECR, use:

    docker-compose push

Your containers are now stored on ECR!