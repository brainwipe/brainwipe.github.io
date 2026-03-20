---
title: "Docker Desktop pull fails on Windows 11 Pro"
date: 2026-03-20
categories: docker microsoft windows terminal
---
If Docker desktop fails to pull like:

```
> docker pull mcr.microsoft.com/dotnet/aspnet:10.0
Error response from daemon: failed to resolve reference "mcr.microsoft.com/dotnet/aspnet:10.0": failed to do request: Head "https://mcr.microsoft.com/v2/dotnet/aspnet/manifests/10.0": EOF
```

or

```
> docker pull public.ecr.aws/docker/library/node:21-alpine3.17
21-alpine3.17: Pulling from docker/library/node failed to copy: httpReadSeeker: failed open: failed to do request: Get [snip hash]
```

Then

- Windows settings
- Network and internet
- Find the network adapter you're using (LAN or Wifi)
- By "More Adapter Options" click "Edit"
- Enter admin password or confirm user access control
- In the Properties window, scroll down to "Internet Protocol Version 6 (TCP/IPv6)" and uncheck.
- Click OK

Then try again and it will work. The docker Hello World pull will work, so it's not all registries.

## Investigation Detail
Windows appears to be preferring ipv6 over ipv4 and when Docker Desktop does a pull, it will not fall back to ipv4 if the ipv6 is not available, it just fails.

The Microsoft and AWS endpoints have slightly different issues but the solution is the same.

### Microsoft Container Registry
As [others have reported](https://github.com/microsoft/containerregistry/issues/165), downgrading to Docker Desktop 4.32.0 (157355) also works. However, our organisation has NCSC Cyber Essentials Plus and we cannot run with outdated software, the downgrade was not an option.

The `nslookup` for the Microsoft Container Registry comes back fine, as does `curl`:

```
nslookup mcr.microsoft.com
Server:  UnKnown
Address:  192.168.4.1

Non-authoritative answer:
Name:    mcr-0001.mcr-msedge.net
Addresses:  2603:1061:f:101::10
          2603:1061:f:100::10
          150.171.69.10
          150.171.70.10
Aliases:  mcr.microsoft.com
          mcr.trafficmanager.net
```

`192.168.4.1` is my Amazon eero gateway router and it acts as a pass through to the WAN. Internally, its DNS is set to `8.8.8.8`, `1.1.1.1`, which is only IPv4 but that should still work. [eero supports ipv6](https://support.eero.com/hc/en-us/articles/115005975026-What-is-IPv6) and before turning off ipv6 `tracert` to an ipv6 address works.

### AWS ECR
AWS recommends that you use images on the [AWS Public Elastic Container Registry (ECR)](https://gallery.ecr.aws/) to keep costs down. In this case, the `nslookup` is:

```
nslookup public.ecr.aws
Server:  UnKnown
Address:  192.168.4.1

Non-authoritative answer:
Name:    a961edf72200aa9b1.awsglobalaccelerator.com
Addresses:  99.83.145.10
          75.2.101.78
Aliases:  public.ecr.aws
```

There's no ipv6 at all! Confused, I tried forcing my ipv6 settings to use the Google IPv6 DNS servers `2001:4860:4860::8888` and `2001:4860:4860::8844`, `nslookup` came back the same (this time using Google's DNS directly):

```
nslookup public.ecr.aws
Server:  dns.google
Address:  2001:4860:4860::8888

Non-authoritative answer:
Name:    a961edf72200aa9b1.awsglobalaccelerator.com
Addresses:  99.83.145.10
          75.2.101.78
Aliases:  public.ecr.aws
```

There's no difference between forcing the DNS to use Google from my PC and the router doing the same anyway.

## This has been a change in the last year
When I was last working with these Docker containers, it all worked without these issues. It is a new problem in the last year or so.