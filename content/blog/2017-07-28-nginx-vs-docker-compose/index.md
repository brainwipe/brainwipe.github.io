---
title: Adding an nginx container for static files in my VS2017 solution
date: 2017-07-28
categories: docker linux vs2017
---
I am using Visual Studio with a docker-compose project to handle the building and running of my docker instances. I want the developer to download the repository, hit F5 and for it to just work. I already have a few linux containers running aspnetcore projects and now I wanted a simple container to serve static files. The static file I needed was a simple HTML page that would call an authentication API. I call it *mcotestclient*.

This is the journey I had and the problems I found.

## Dockerfile for the new container
I decided to use a super-light linux container (which is inline with the brief of our solution) and decided on [nginx](https://www.nginx.com/resources/wiki/) as the web server to hand the static files. To add a new container, I began with a new solution folder with an HTML file (`index.html`) and a Dockerfile:

    FROM nginx:alpine
    COPY . /usr/share/nginx/html

This Dockerfile gets the nginx docker image, which is in turn built upon a super lightweight [Alpine](https://alpinelinux.org/) Linux distribution. It then copies the contents of the folder (just my boring `index.html`) into `/user/share/nginx/html`, which is the default location.

The Dockerfile only creates a new image, to have a new instance run, we need to add a new service to the `docker-compose.yml` file.

## Adding a new container to docker-compose
In my docker-compose project, I opened `docker-compose.yml` and added a new image:

    services:
        [snip my other containers]

        mcotestclient:
            image: mcotestclient
            build: 
                context: ./Client
                dockerfile: Dockerfile
            ports:
            - 56108:80
            networks:
            - mconetwork

`mco` is our company prefix, the folder where `index.html` and the Dockerfile was located is `./Client` and I want it to run on port `56108` so that when I go to `http://localhost:56108` I see my index page.

## First error: Can not find label com.microsoft.visualstudio.targetoperatingsystem
I built the solution and got the following:

    Error	MSB4018	The "PrepareForBuild" task failed unexpectedly.
    System.InvalidOperationException: Can not find label 'com.microsoft.visualstudio.targetoperatingsystem'.
    at Microsoft.DotNet.Docker.DockerServiceDebugProfileProvider.GetLabelValue(IReadOnlyDictionary`2 labels, String labelName, Boolean required)
    at Microsoft.DotNet.Docker.DockerServiceDebugProfileProvider.ParseDockerServiceDebugProfiles(String workspaceName, DockerComposeDocument document)
    at Microsoft.DotNet.Docker.DockerWorkspace.<GetDockerServiceDebugProfilesAsync>d__12.MoveNext()
    --- End of stack trace from previous location where exception was thrown ---
    at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
    at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
    at Microsoft.DotNet.Docker.DockerWorkspace.<PrepareForBuildAsync>d__13.MoveNext()
    --- End of stack trace from previous location where exception was thrown ---
    at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
    at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
    at Microsoft.DotNet.Docker.BuildTasks.DockerBaseTask.Execute()
    at Microsoft.Build.BackEnd.TaskExecutionHost.Microsoft.Build.BackEnd.ITaskExecutionHost.Execute()
    at Microsoft.Build.BackEnd.TaskBuilder.<ExecuteInstantiatedTask>d__26.MoveNext()	docker-compose	C:\Program Files (x86)\Microsoft Visual Studio\2017\Professional\MSBuild\Microsoft\VisualStudio\v15.0\Docker\Microsoft.VisualStudio.Docker.Compose.targets	153	

The most important line is that `Can not find label 'com.microsoft.visualstudio.targetoperatingsystem'`. This means that Visual Studio needs to know what the operating system is. In your docker-compose project, click the little triangle next to `docker-compose.yml` and you'll see that there are a number of additional files that Visual Studio needs. Open `docker-compose.vs.debug.yml`. I needed to put in a new entry in there to tell it what the target operating system was for my new image:

    mcotestclient:
        image: mcotestclient:dev
        labels:
        - "com.microsoft.visualstudio.targetoperatingsystem=linux"

## Second Error - no such file or directory /bin/bash
For the second error, the Error window showed that: 

    Error	MSB4018	The "PrepareForBuild" task failed unexpectedly.
    Microsoft.DotNet.Docker.CommandLineClientException: .

    For more troubleshooting information, go to http://aka.ms/DockerToolsTroubleshooting ---> Microsoft.DotNet.Docker.CommandLineClientException
    at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
    at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
    at Microsoft.DotNet.Docker.DockerClient.<ExecuteAsync>d__0.MoveNext()
    --- End of inner exception stack trace ---
    at Microsoft.DotNet.Docker.DockerClient.<ExecuteAsync>d__0.MoveNext()
    --- End of stack trace from previous location where exception was thrown ---
    at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
    at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
    at Microsoft.DotNet.Docker.DockerWorkspace.<PrepareForBuildAsync>d__13.MoveNext()
    --- End of stack trace from previous location where exception was thrown ---
    at System.Runtime.CompilerServices.TaskAwaiter.ThrowForNonSuccess(Task task)
    at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
    at Microsoft.DotNet.Docker.BuildTasks.DockerBaseTask.Execute()
    at Microsoft.Build.BackEnd.TaskExecutionHost.Microsoft.Build.BackEnd.ITaskExecutionHost.Execute()
    at Microsoft.Build.BackEnd.TaskBuilder.<ExecuteInstantiatedTask>d__26.MoveNext()	docker-compose	C:\Program Files (x86)\Microsoft Visual Studio\2017\Professional\MSBuild\Microsoft\VisualStudio\v15.0\Docker\Microsoft.VisualStudio.Docker.Compose.targets	153	

This is Visual Studio telling us that something happened on the command line and it doesn't know how to deal with it. In this case, go to the Visual Studio Output window, ensure you've got Build selected in the drop down at the top and scroll up to the line about that stack trace, you'll see the actual commands it was trying to run:

    [snip]
    docker  ps --filter "status=running" --filter "name=dockercompose320715364_mcotestclient_" --format {% raw %}{{.ID}}{% endraw %} -n 1
    2>ab666c689e17
    2>docker  exec -i ab666c689e17 /bin/bash -c "if PID=$(pidof -x dotnet); then kill $PID; fi"
    2>rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:262: starting container process caused "exec: \"/bin/bash\": stat /bin/bash: no such file or directory"
    2>C:\Program Files (x86)\Microsoft Visual Studio\2017\Professional\MSBuild\Microsoft\VisualStudio\v15.0\Docker\Microsoft.VisualStudio.Docker.Compose.targets(153,5): error MSB4018: The "PrepareForBuild" task failed unexpectedly.
    2>C:\Program Files (x86)\Microsoft Visual Studio\2017\Professional\MSBuild\Microsoft\VisualStudio\v15.0\Docker\Microsoft.VisualStudio.Docker.Compose.targets(153,5): error MSB4018: Microsoft.DotNet.Docker.CommandLineClientException: .
    [snip]

At the boittom I've included the first two lines of the stack trace to show you what it looks like. The docker-compose project uses MSBuild to run the docker commands for you. At the top you can see that it first needs the process ID of an existing *mcotestclient* container, which is `ab666c689e17` and then it passes it to an `exec` command. That exec command attaches to the running container and kills the dotnet process inside (there isn't one but it doesn't know that). However, this fails. The next line says that the exec failed because `/bin/bash` does not exist. That folder isn't there because bash isn't there.

**BASH NOT EXIST? BUT THIS IS LINUX!**

No, this is Alpine Linux, which doesn't have bash. The Visual Studio compose file assumes that every Linux container has bash on it. Not unreasonable but not right.

### Options: install bash or change the base image
A change needs to be made to my super light Dockerfile, which currently looks like this:

    FROM nginx:alpine
    COPY . /usr/share/nginx/html

Either add bash to Alpine like this:

    FROM nginx:alpine
    RUN apk update && apk add bash
    COPY . /usr/share/nginx/html

Or choose the default container:

    FROM nginx
    COPY . /usr/share/nginx/html

As I wanted simplicity and that this would not be a production container, I went with the default container.

**NB: Make sure you clean, rebuild and run the solution, else the change will not be picked up!**