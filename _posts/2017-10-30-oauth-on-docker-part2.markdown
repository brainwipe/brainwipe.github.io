---
layout: post
title: "ASP.NET Core 2 API on Docker with OAuth (Part 2)"
date: 2017-10-30 10:00:00 +01:00
categories: docker dotnet oauth identityserver
---
In [Part 1]({% post_url 2017-10-30-oauth-on-docker-part1%}) we built an ASP.NET Core 2 API and got an Identity Server all running on docker containers. In this part we're going to add a client application that can get a token from the Identity Server, apply authorization to the API service and then use the token to call the service.

You can grab the completed code from the [DockerDotNetOAuth](https://github.com/brainwipe/DockerDotNetOAuth) GitHub repository.

## Specify Port Binding in Docker Compose
Up until now we've let docker compose decide what ports we're going to use. This has been fine so far but we need to fix them so that the client HTML page knows where its resources are. The best solution is to use domains and forwarding but that's out of the scope of this post.

Open `docker-compose.yml` and update the existing services to include the port numbers:

```yml
services:
  dockerdotnetoauth:
    image: dockerdotnetoauth
    ports:
      - 32771:80
    build:
      context: ./DockerDotNetOAuth
      dockerfile: Dockerfile

  identityserver:
    image: identityserver
    ports:
      - 32772:80
    build:
      context: ./IdentityServer
      dockerfile: Dockerfile
```

This is the same port binding that we saw in the `docker ps` command. Now with subsequent runs of the system, the ports will remain the same.

## Client Docker Container
For our client, we're going to use an HTML web page running on an [nginx docker container](). We'll get the client container running first and then come back for the HTML.

In your solution folder, create a new folder called Client. You'll need to do the same in Visual Studio. In the Client folder, add a new file called `Dockerfile` (no extension). The Dockerfile is going to describe how to build our client container image. The dockerfile is extremely simple:

```yml
FROM nginx
EXPOSE 32773
```

For our system to turn this Dockerfile description into a container image, we need to add it to our `docker-compose.yml` file. Add the following section into your docker-compose file under your other services. Make sure that the `testclient` is aligned with the pre-existing `identityserver:` line.

```yml
testclient:
    image: testclient
    build: 
      context: ./Client
      dockerfile: Dockerfile
    volumes:
      - ./Client:/usr/share/nginx/html
    ports:
     - 32773:80
``` 

This will use the Dockerfile found in the `./Client` folder and bind external port `32773` to the nginx default of port `80`. You can run the solution at this point and the container will run but there won't be any content at `http://localhost:32773/` because we've not make the HTML yet, nginx will give a 403 by default.

### Volumes
Different to our other containers is `volumes`, this line maps a folder on our hard-drive with a folder in the running container. We want to do this so that when we update the client HTML file (we're about to build) then the container updates automatically. This saves us from having to rebuild the container image every time we make an HTML change. This is useful for the client as it's only a single HTML page that doesn't have a build process. 

If you are using .NET or webpack or npm or any modern front end tooling you will have a build chain whose output can be put into the container image on each build. The benefit of copying the build output into the container image is that then the resulting container contains *everything you need to run the site*. You can deploy that image to test, staging and so on. If you use volumes then you are depending on files outside the container.

## Create simple HTML client
We're going to use jQuery to get a token from Identity Server. Add a new HTML file called `index.html` in `./Client/`. Here's the source:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test Client</title>
</head>
<body>
Test Client
<script src="https://code.jquery.com/jquery-3.2.1.min.js"/></script>
<script>
    function GetToken() {
        $.ajax({
                type: 'POST',
                url: 'http://localhost:32772/connect/token',
                crossDomain: true,
                timeout: 2000,
                data: {
                    "client_id": "client",
                    "grant_type": "client_credentials",
                    "client_secret": "secret",
                    "scopes": "api1"
                }
            })
            .done(function(data) {
                console.log("Got token: " + data.access_token);
            });
    }

    $(function () {
        GetToken();
    });
</script>
</body>
</html>
```

Once the page has loaded then a POST Ajax query is sent to the token creation endpoint of Identity Server with the scope `api1`, secret of `secret`, client name of `id` and a grant type of `client_credentials`, which simply means that a secret is all that's required. If it is successful then it will print the token to the javascript console.

In Visual Studio Press <kbd>F5</kbd> to see it fail!

![We haven't set up CORS yet]({{site.url}}/assets/aspnet-core2-cors.png)

## Setting up CORS on Identity Server
The error above is telling us that the server will not accept POST requests from our origin:

```
Failed to load http://localhost:32772/connect/token: No 'Access-Control-Allow-Origin' header is present on the requested resource. 
Origin 'http://localhost:32773' is therefore not allowed access. The response had HTTP status code 400.
```

As the Identity Server and client are hosted on different ports, they are different origins, so this is a Cross Origin request and the Identity Server needs to have [Cross Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) enabled.

First we're going to "turn off" ASP.NET's CORS system as Identity Server can do it with much more granularity. Open `startup.cs` and add the following line into the `Configure(...)` method.

```cs
app.UseCors(builder =>builder.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod().AllowCredentials());
```

Next we're going to tell Identity Server to use an in memory CORS policy. Update your `ConfigureServices(...)` method to:

```cs
services.AddIdentityServer()
    .AddInMemoryApiResources(Configuration.GetApiResources())
    .AddInMemoryClients(Configuration.GetClients())
    .AddCorsPolicyService<InMemoryCorsPolicyService>() // Add the CORS service
    .AddDeveloperSigningCredential();
```

Finally we need to tell Identity Server which origins are allowed. Open `Configuration.cs` and add the following under `AllowedScopes`:

```cs
AllowedCorsOrigins = new[] {"http://localhost:32773"}
```

This allows only the test client to request tokens. Run the app <kbd>F5</kbd>, point your browser at your [test client](http://localhost:32771), open the console and you should see the token that we're going to pass to the API. It will look something like this:

```
Got token: eyJhbGciOiJSUzI1NiIsImtpZCI6IjUzZTVjZDE3YWEzYjkxZGUwMmUyZWI5MzdiODdmZTU2IiwidHlwIjoiSldUIn0.eyJuYmYiOjE1MDkzNzQ1MzAsImV4cCI6MTUwOTM3ODEzMCwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDozMjc3MiIsImF1ZCI6WyJodHRwOi8vbG9jYWxob3N0OjMyNzcyL3Jlc291cmNlcyIsImFwaTEiXSwiY2xpZW50X2lkIjoiY2xpZW50Iiwic2NvcGUiOlsiYXBpMSJdfQ.DkS9Wcb3AzvzcCJS1JnBICE9pBuzc4d8j1LzbUcgyjKbha56c3NSNyT6W0yZSm5LA5aaBbMVJMnZYYrfbCGDUkbKbX2NmKfkrB16O0y5BH2z1ruBXjMUG2Iks36VYbsGS8ZBd7InTmPeYk4hdjygDkLyFlH5WI94_KKYIE5wPQNVH2d7iiQgocKI8WlIphCCLjPA5CG-kyTYl8zUtT9lxh1fJcXtjvZKphfpbPijgw5_l0fZWvnlk4GS0Xi9bvsFxwATV3ISw_geUCKSWV5sojFCnIyeCgtlmGXFUvetUvVB1xVpuRN-JU1LVGgT1Fx3kszLdwip_wahu-CnooOcpA
```

## Setting CORS on the API
Before we add authorization onto the API, we must tell the API that the client site is allowed to access it. We do this with CORS but unlike we did in [Part 1]({% post_url 2017-10-30-oauth-on-docker-part1%}), we want ASP.NET to control access rather than Identity Server. 

Open `startup.cs`, in `ConfigureServices(...)` add:

```cs
services.AddCors();
```

In `Configure(...)` add *before* `UseMvc()`:

```cs
app.UseCors(
    builder => builder
        .AllowAnyHeader()
        .AllowAnyMethod()
        .WithOrigins("http://localhost:32773"));
```

This is saying that it will allow any header value, any method (POST, GET, PUT etc) from the client website. Before production we would limit this to just the methods and headers we want.

## Setting Issuer Uri on Identity Server
When we run Identity Server as a developer on our local boxes, we map the port such that we can simply go to `http://localhost:32772/`. The API needs to be able to contact the Identity Server to get the discovery document and check the client's token then it needs a URI to call. The default Issuer Uri for the Identity Server is also `http://localhost:32772`. You can double check this on the discovery document at [localhost:32772/.well-known/openid-configuration](localhost:32772/.well-known/openid-configuration).

Our system is built within docker containers, so when the API calls `localhost:32772` then it will look at port `32772` of its own container. So the API will need to use `http://identityserver` instead (we'll set that in the next step). However this will not work straight away because the Issuer Uri in Identity Server and the Authority used by the API need to match. Therefore we need to override the Issuer Uri in Identity Server to `http://identityserver`.

Open the Identity Server `Startup.cs` and in `ConfigureServices(...)` change the `AddIdentityServer()` line to:

```cs
services.AddIdentityServer(opt => opt.IssuerUri = "http://identityserver")
```
When you now call the discovery document on [localhost:32772/.well-known/openid-configuration](localhost:32772/.well-known/openid-configuration) then you'll see the Issuer Uri is `"http://identityserver"`.

### This is simple but not ideal
This works for our example but the best solution (not covered here) is to assign domains to your containers. If we had an API external to your docker host then it would not be able to validate tokens against `http://identityserver` because that URI is only understood inside your docker host.

## Securing the API Service
In this step we're going to secure our `api/value` web service so that it requires a bearer token. First we're going to install the [IdentityServer4.AccesstokenValidation](https://www.nuget.org/packages/IdentityServer4.AccessTokenValidation/) nuget package into our API project. 

Open `Startup.cs`, in `ConfigureService(...)` add:

```cs
services.AddAuthentication(opt =>
    {
        opt.DefaultScheme = IdentityServerAuthenticationDefaults.AuthenticationScheme;
        opt.DefaultAuthenticateScheme = IdentityServerAuthenticationDefaults.AuthenticationScheme;
    })
    .AddIdentityServerAuthentication(
        opt =>
        {
            opt.Authority = "http://identityserver";
            opt.RequireHttpsMetadata = false;
            opt.ApiName = "api1";
        });
```

The first call to services (`AddAuthentication`) sets the authentication scheme to `bearer` and then the identity server is given as the secure token server. We don't need the secret at this point because we're not a client. We will take the token from the client and pass it to the Identity Server to check.

In the `Configure(...)` method, add authentication to pipeline *before* `UseMvc()`:

```cs
app.UseAuthentication();
```

Finally we need to specify which controllers need authentication. Open `./Controllers/ValuesController.cs` and add the `Authorize` attribute to the controller like so:

```cs
    [Route("api/[controller]")]
    [Authorize]
    public class ValuesController : Controller
    {
        // Snip for brevity
```

## Test without the token
Before we start using the token, press <kbd>F5</kbd> and the sites will run up. If you go to `http://localhost:32771/api/values` then nothing will appear, in the browser's console (press F12), the network tab will show `401 Unauthorized`. Identity Server isn't being called at all, though because there is no token supplied so ASP.NET responds automatically.

## Calling the Values Service with a token
We're now going to call the `api/values` service from our test client using the token we already have.

Open `./Client/index.html` and add this function inside the `<script>` tag:

```js
function CallService(token) {
    $.ajax({
            type: 'GET',
            url: 'http://localhost:32771/api/values',
            crossDomain: true,
            timeout: 2000,
            beforeSend: function(xhr) { xhr.setRequestHeader('Authorization','Bearer ' + token) }
        })
        .done(function(data) {
            console.log(data);
        });
}
```

This function calls the values API, adding in a Bearer token as the authorization header. For simplicity we're going to call this from the `done` promise of the `GetToken()` call we had before:

```js
    .done(function(data) {
        console.log("Got token: " + data.access_token);
        CallService(data.access_token);
    });
```

When you reload the test client page [http://localhost:32773/](http://localhost:32773/) then the client first authenticates against Identity Server to get a token and that token is used in the `Authorization` header to call the values API. We have a success! I looks like:

![Chrome console has both the token and the values from the API]({{site.url}}/assets/aspnet-core2-result.png)

We have the token printed first and then the result from the API call.

## Well done for making it this far
If you jumped in at this point, check out [Part 1]({% post_url 2017-10-30-oauth-on-docker-part1%}) and you can grab the code from the [DockerDotNetOAuth](https://github.com/brainwipe/DockerDotNetOAuth) GitHub repository.