---
title: IIS configuration for Single Page Apps
summary: A guide for setting up Internet Information Services (IIS) to serve a Single Page Application (SPA)
description: A guide for setting up Internet Information Services (IIS) to serve a Single Page Application (SPA)
date: "2022-02-04"
tags:
  - IIS
  - SPA
  - router
  - cache
---

This is a short guide for setting up an Internet Information Services (IIS) site that serves a Single Page Application (SPA).

There are three obstacles to overcome here:

1. How to serve a SPA with client-side routing
2. How to serve a non-cached, always up to date version of the SPA
3. How to prevent IIS from shutting down automatically

Each problem will be addressed in this post, with additional information below the config. Here's the final web.config we will create:

{{< code language="xml" title="web.config" isCollapsed="true" expand="Show" collapse="Hide" >}}

<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <rule name="redirect to index" stopProcessing="false">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll">
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                        <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="/" />
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
    <location path="index.html">
        <system.webServer>
            <httpProtocol>
                <customHeaders>
                    <add name="Cache-Control" value="no-cache, no-store" />
                </customHeaders>
            </httpProtocol>
        </system.webServer>
  </location>
</configuration>
{{< /code >}}

## Serve static files

In contrast to a pure static file server, we need IIS to always serve `index.html`, except for file requests that contain our JavaScript, CSS or other assets. This way we also support client-side routing.

First, install the [URL Rewrite](https://www.iis.net/downloads/microsoft/url-rewrite) extension.
Then, add the file `web.config` to the folder that contains the SPA files (aka the folder with `index.html`). This `web.config` contains all the IIS settings for your site.
Simply add the necessary rewrite rules:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <rule name="redirect to index" stopProcessing="false">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll">
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                        <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="/" />
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
</configuration>
```

<details>
<summary>See detailed explanation</summary>

File servers generally serve only the files on disk, which means there is a 1:1 mapping between a file request from the browser and the file on disk on the server. A user visiting the root of our SPA will therefore see the site as expected, because all the static files like `index.html` and the referenced scripts and assets really exist on the server.

But when using client side routing, like Angular Router or React Router, you'll run into issues quickly.
Imagine you configured a route with the path `/products/:id` in your router. If the user visits a product with ID 1 directly (not from inside the app), IIS tries to find a file called `1` inside a folder `products`. However, this file will never exist, so IIS returns an error and the user will see the message **404 - Not Found**.

To fix this, we need IIS to return our `index.html` whenever we don't request an actual file or directory. The rewrite extension enables us to specify these exact rules with just a few lines of XML.

</details>

## Keeping users up to date

The entrypoint to the app, `index.html`, should never be cached by the browser. Once you upload a new version to the server, all users with a cached version will either see no changes at all, a blank screen or errors.
To prevent any form of caching we add a header for `Cache-Control` with the value `no-cache, no-store` when serving `index.html`. All other files may be cached.

```xml
<configuration>
    [...]
    <location path="index.html">
        <system.webServer>
            <httpProtocol>
                <customHeaders>
                    <add name="Cache-Control" value="no-cache, no-store" />
                </customHeaders>
            </httpProtocol>
        </system.webServer>
    </location>
</configuration>
```

<details>
<summary>Detailed explanation</summary>

To explain this behavior, we must take a look at how build tools like _Angular CLI_ and _Create React App_ generate files :

> Each file inside of the build directory will have a unique hash appended to the filename that is generated based on the contents of the file, which allows you to use aggressive caching techniques to avoid the browser re-downloading your assets if the file contents haven't changed.
> If the contents of a file changes in a subsequent build, the filename hash that is generated will be different.
>
> **- [Create React App Docs](https://create-react-app.dev/docs/production-build/)**

You can test this very easily on your own. Simply build your app, make changes to it and build again to a different folder afterwards. As you can see, updating the app results in a different set of files. Only `index.html` always keeps its name, but its contents reference the updated files. Remember: `index.html` is the entrypoint to your app. This file specifies all the scripts and styles necessary to kickoff rendering the whole site.

Imagine a user visited your site and as a result now has the following files cached locally:

- `index.html`
- `main.aaa.js`
- `styles.111.css`

After a while, you publish another version with updated styles and bug fixes. Let's assume the build tool produces these files:

- `index.html`
- `main.bbb.js`
- `styles.222.css`

Now, since the users browser (or some proxy server) has no way of telling when the cached file is outdated this user is stuck on the older version. Adding the cache-control header to your config as seen above forces the browser to never cache `index.html` and always request it from the server. And because `index.html` references all other hashed files the browser will load them as well. As a result, the user will always have the newest version of your app.

As mentioned above, all hashed files can even be cached indefinitely. At some point, the users browser could have cached multiple versions of our app, like so:

- `index.html`
- `main.aaa.js` (cached, not used)
- `main.bbb.js` (cached)
- `styles.111.css` (cached, not used)
- `styles.222.css` (cached)

Luckily, we're now using our hashed files the right way and don't really care what else the browser stored in the past.

IIS already sends lots of useful headers out of the box: you'll notice the headers _Date_, _Last-Modified_, and _ETag_ when debugging requests. Browsers perform a lot of "guess work" when we don't exactly specify how to cache things, so expect different behavior depending on the vendor and version.
If you want to know more about caching concepts in general I recommend [this web.dev article](https://web.dev/http-cache/).

</details>

## Prevent IIS from shutting down

If you require your site to be available at all times, then these settings could help you deliver consistent response times.
To be clear, the site or IIS itself can't shut down automatically. The worker processes that manage the requests, however, can. That's why we have to adjust these settings in the IIS manager:

![IIS application pool settings](/img/IIS_application_pool_settings.png)

First, select the _application pool_ for your site, then open the advanced settings.

**Idle Time-out**

Setting _Idle Time-out_ to _0_ might already be enough for your use case. The default setting is _20_, which means that IIS would kill the worker process after 20 minutes of inactivity to free resources. Setting it to _0_ ensures that IIS keeps the process around to instantly handle incoming requests.

**Start mode**

To make sure a worker process is available to begin with, you can also set _Start mode_ to _AlwaysRunning_. The default setting is _OnDemand_, which means a worker process is only created once the very first request to the site is received. With _AlwaysRunning_ the process is created right away, which further reduces initial response times.

As always, try to profile your system to understand which settings provide the best balance between availability and total resource load.
