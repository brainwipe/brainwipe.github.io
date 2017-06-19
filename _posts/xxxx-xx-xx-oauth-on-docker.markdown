---
layout: post
title: "Adding OAuth authentication to ASP.NET Core running on docker"
date: xxxx-xx-xx 13:00:00 +01:00
categories: docker dotnet oauth
---
I previously explained how to get a [ASP.NET Core site running on docker]({% post_url 2017-05-23-docker-linux-dotnet %}) in this post we're going to add a second container that runs an authentication server. I am going to use the free, excellent [Identity Sever 4](https://identityserver.io/) to provide the OAuth authentication functionality.

## Why run a separate container?

As the examples in the Identity Server documentation show, you can run Identity Server [in your existing ASP.NET Core MVC project](https://identityserver4.readthedocs.io/en/release/quickstarts/0_overview.html). However, containerisation gives us the ability to split up our application into separate micro services. In this way we can choose to release API or authentication server (or both).