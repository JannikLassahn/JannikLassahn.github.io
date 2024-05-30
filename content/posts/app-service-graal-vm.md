---
title: Using executables on Azure App Service for Windows
summary: Run GraalVM native images (or any other executable) on Azure App Service
description: Run GraalVM native images (or any other executable) on Azure App Service
date: "2024-05-30"
tags:
  - Azure
  - App Service
  - Java
  - GraalVM
---

This post describes how to deploy executables (*.exe*) on Azure App Service for Windows. The method shown here uses GraalVM native images, but works for all kinds of languages and frameworks:
1. Build a server executable (with support for external port configuration)
2. Create web.config (with HttpPlatformHandler)
3. Upload the files to *site\wwwroot* and restart App Service

### Motivation

App Service integrates nicely with commmon platforms like ASP.NET, NodeJS and Java. But what if you want to deploy your Go, Rust or [insert latest trendy language] app that isn't supported out of the box? Well, you're in luck, because the integration is usually quite easy.

My use case is more experimental: I'm using Java with Spring, which can eat up lots of precious system resouces, especially RAM. I wanted to try out an alternative, and potentially more performant, approach using AOT compilation.

### 1. Build the executable (GraalVM)


[GraalVM](https://www.graalvm.org/) describes itself as "an advanced JDK with ahead-of-time Native Image compilation" which claims, among other things, drastically lower resource usage, faster startup and enhanced security.

For starters, I [installed the GraalVM community edition](https://www.graalvm.org/latest/docs/getting-started/windows/), cloned their [examples repository](https://github.com/graalvm/graalvm-demos) and built the example for *Spring*:  
```
git clone https://github.com/graalvm/graalvm-demos.git
cd graalvm-demos\spring-native-image
mvnw native:compile -Pnative
```
Running `target\benchmark-jibber-snapshot-0.0.1-SNAPSHOT.exe` locally starts the server as expected, so let's move on. 

### 2. Create web.config

The `web.config` is used to configure IIS such that:
- the `HttpPlatformHandler` module handles all requests
- the module calls the executable and assigns a port

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <handlers>
            <add name="httpplatformhandler" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified" />
        </handlers>
        <httpPlatform processPath="D:\home\site\wwwroot\azureapp.exe" 
                      arguments="-Dserver.port=%HTTP_PLATFORM_PORT%">
        </httpPlatform>
    </system.webServer>
</configuration>
```

Let's examine the configuration of `httpPlatform`:

**processPath** holds the path to your executable. The actual file name doesn't matter. In this case I renamed my executable to `azureapp.exe`, because this was the default name for self contained [Go applications](https://azure.microsoft.com/de-de/blog/running-go-applications-on-azure-app-service/) back in the day.

**arguments** holds command line arguments for the executable. Also a fitting place to tell our server which port the App Service wants us to use. The port will be in the environment variable `HTTP_PLATFORM_PORT`. In case of Spring, we can simply use the property `server.port` to assign the aforementioned variable. 

Even if your executable does not suppport this kind of external configuration, you should at least be able to read environment variables like `HTTP_PLATFORM_PORT` from within the application code.

Additional attributes like **stdoutLogEnabled** and **stdoutLogFile** might be helpful during initial setup.

For more information about the `HttpPlatformHandler` module see the  [documentation](https://learn.microsoft.com/en-us/iis/extensions/httpplatformhandler/httpplatformhandler-configuration-reference).

### 3. Upload to App Service

All that is left to do is upload all artifacts to *site\wwwroot* using your preferred deployment mechanism, e.g. with the Kudu UI or Azure CLI, and restart the App Service if necessary. 

You might have to adjust additional server properties. For example:
the first thing I had to fix was the maximum request header size. Since I enabled authentication the request header was too large for the default embedded server (Tomcat). To solve this issue, I added `-Dserver.max-http-request-header-size=1MB` to httpPlatform arguments.
