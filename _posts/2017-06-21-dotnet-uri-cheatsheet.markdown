---
layout: post
title: ".NET Uri class cheatsheet"
date: 2017-06-21 09:00:00 +01:00
categories: dotnet aspnet cheatsheet
---
The [Uri](https://msdn.microsoft.com/en-us/library/system.uri.aspx/) class in .NET contains methods for getting at parts of the Uri. In your ASP.NET MVC project you can get the current URL from within a Controller using:

     Uri currentUrl = Request.Url;

From this object you can then get lots of information about the current URL. I find that each time I use it, I have to look up which method gives what. Here is a quick cheatsheet for the methods and properties:

For the URL: [https://github.com/brainwipe/NJsonApiCore/search?utf8=%E2%9C%93&q=jsonapi&type=](https://github.com/brainwipe/NJsonApiCore/search?utf8=%E2%9C%93&q=jsonapi&type=)

| Uri Property   | Value                                                                           |
|----------------|---------------------------------------------------------------------------------|
| AbsolutePath   | /brainwipe/NJsonApiCore/search                                                  |
| AbsoluteUri    | https://github.com/brainwipe/NJsonApiCore/search?utf8=%E2%9C%93&q=jsonapi&type= |
| Authority      | github.com                                                                      |
| DnsSafeHost    | github.com                                                                      |
| Fragment       |                                                                                 |
| Host           | github.com                                                                      |
| HostNameType   | Dns                                                                             |
| IsAbsoluteUri  | True                                                                            |
| IsDefaultPort  | True                                                                            |
| IsFile         | False                                                                           |
| IsLoopback     | False                                                                           |
| IsUnc          | False                                                                           |
| LocalPath      | /brainwipe/NJsonApiCore/search                                                  |
| OriginalString | https://github.com/brainwipe/NJsonApiCore/search?utf8=%E2%9C%93&q=jsonapi&type= |
| PathAndQuery   | /brainwipe/NJsonApiCore/search?utf8=%E2%9C%93&q=jsonapi&type=                   |
| Port           | 443                                                                             |
| Query          | ?utf8=%E2%9C%93&q=jsonapi&type=                                                 |
| Scheme         | https                                                                           |
| Segments       | / brainwipe/ NJsonApiCore/ search                                               |
| UserEscaped    | False                                                                           |
| UserInfo       |                                                                                 |

[C# .net console gist](https://gist.github.com/brainwipe/845a24fe773186b318a91e51b8b2a916) for generating this output and a [handy Markdown table builder](http://www.tablesgenerator.com/markdown_tables#).