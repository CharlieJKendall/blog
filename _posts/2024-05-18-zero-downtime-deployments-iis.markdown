---
layout: post
title: "Simple Zero Downtime Deployments For .NET Applications on IIS"
date: 2024-05-18
tags: [ 'IIS', 'zero-downtime', 'deployments', '.NET' ]
summary: 'It is now easier than ever to implement a simple strategy for deploying your applications to IIS without any downtime using the new ASP.NET Core Hosting Bundle shipping with .NET 9.'
---

System downtime is rarely something that is met with smiles and laughter, least of all when you're __deploying__ an application - yet 'accepted' downtime is surprisingly commonplace among enterprise software companies! While it's true that modern tooling like Kubernetes and managed services like Azure App Service have made zero downtime deployments all but trivial, many companies do not have these options readily available to them.

So... you're probably reading this because you're one of the lucky folks who deploy your .NET applications to IIS. You may have already read some blogs about how to achieve zero downtime deployment with blue/green deployments, server farms, and a scattering of PowerShell scripts which got you thinking "hmm... that seems like a decent chunk of work and maintenance overhead, maybe another time". Well fear no more, because as of April 2024 we have an far simpler option involving just two steps!

## Everybody loves a quick win

1. Install the [latest](https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/runtime-aspnetcore-8.0.10-windows-hosting-bundle-installer) ASP.NET Core Hosting Bundle on your server(s) running IIS
2. If your application is targetting .NET 8 or earlier, add the following to your web.config file to opt in to the fix that enables overlapped app pool recycling (.NET 9 onwards has this enabled by default)

``` xml
<aspNetCore processPath="dotnet" arguments="your_application.dll" stdoutLogEnabled="false" stdoutLogFile=".logs_stdout">
  <handlerSettings>
    <!-- This is the golden ticket: -->
    <handlerSetting name="shutdownDelay" value="1000" />
  </handlerSettings>
</aspNetCore>
```

3. Ensure your deployment pipeline copies the new application artifacts to a separate location on the filesystem to your existing application artifacts
4. When you are ready to go, change the physical path of your live website/application to point at the new artifact directory (this is straightforward with [appcmd.exe](https://learn.microsoft.com/en-us/iis/get-started/getting-started-with-iis/getting-started-with-appcmdexe))

Easy! If you want some additional context or to dig deeper into why this wasn't an option before then check out the links below:

- [https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/advanced#reduce-503-likelihood-during-app-recycle](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/advanced#reduce-503-likelihood-during-app-recycle)
- [https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-9.0?view=aspnetcore-8.0#fix-for-503s-during-app-recycle-in-iis](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-9.0?view=aspnetcore-8.0#fix-for-503s-during-app-recycle-in-iis)
- [https://github.com/dotnet/aspnetcore/pull/52807](https://github.com/dotnet/aspnetcore/pull/52807)
