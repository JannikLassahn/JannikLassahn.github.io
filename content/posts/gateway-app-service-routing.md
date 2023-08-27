---
title: Azure App Service routing using Application Gateway
summary: Use Azure Application Gateway to give multiple App Services the same domain and route between them using path based rules
description: Use Azure Application Gateway to give multiple App Services the same domain and route between them using path based rules
date: "2023-08-26"
tags:
  - Azure
  - App Service
  - Application Gateway
---

Requirement:

have multiple App Services accessible on the **same domain**, but **different URL paths**.

Solution:

since one cannot bind the same custom domain to multiple App Services, we use Azure Application Gateway to handle all requests to the custom domain and configure path rules to route to respective App Services.

<!-- experiment with themed SVG -->

<object id="gw-overview" data="/img/appservice-appgw/gw-overview.svg" style="max-width:100%;display:block"></object>

<script>
  let svg;

  const observer = new MutationObserver((mutations => {
    mutations.forEach((mutation) => {
      if (
        mutation.type === 'attributes' &&
        mutation.attributeName === 'class'
      ) {
        svg.classList.toggle("dark-theme", document.body.classList.contains("dark-theme"));
      }
    });
  }));


  const obj = document.getElementById("gw-overview");
  obj.addEventListener("load", () => {
    svg = obj.contentDocument.rootElement;
    if(document.body.classList.contains("dark-theme")) {
      svg.classList.add("dark-theme");
    }
    observer.observe(document.body, { attributes: true });
  })

  
</script>

## Example setup

The following example shows how to configure the solution in detail. In addition to the App Services seen above, we'll have an App Service for handling remaining traffic; consider it a "fallback" route.

### Prerequisites

- 3 App Services
  - "Service A" (gw-service-a.azurewebsites.net)
  - "Service B" (gw-service-b.azurewebsites.net)
  - "Service Default" (gw-service-default.azurewebsites.net)
- custom domain and respective PFX certificate

### Create Application Gateway

When creating the gateway, you'll have to go through an initial step-by-step wizard.
Make sure to fill out the "Basics" tab according to your requirements and add a public IP address in the "Frontends" tab.

> Please read the [documention](https://learn.microsoft.com/en-us/azure/application-gateway/) for more information about various concepts/abstractions of App Gateway.

#### Backend pools

In the "Backends" tab add a backend pool for each App Service. The following screenshot shows the pool for App Service "A".

![Backend pool](/img/appservice-appgw/gw-add-backend-pool.jpeg)

#### Rules

Moving on to the "Configuration" tab, we have to connect the frontend and backend.
Click on "Add routing rule" to open the rule form. It might be overwhelming at first, since it is not only used to add rules, but listeners and backend settings as well.

![Rule](/img/appservice-appgw/gw-add-https-rule.jpeg)

For now, we'll setup the gateway to handle HTTPS traffic and route it to the backend pools.
If you don't want to setup HTTPS in the gateway, remember to disable HTTPS redirection in the App Services.

- choose any name and priority
- in the "Listener" tab
  - set protocol to HTTPS
  - upload a certificate for the desired domain and enter the password
- in the "Backend targets" tab
  - set backend target to the "fallback" backend pool
  - click "Add new backend settings"

##### Rules -> Add backend settings

The backend settings contain the most important settings when using App Services:

![Rule](/img/appservice-appgw/gw-add-backend-settings.jpg)

**Override backend path** changes the path that the App Service receives. If you leave it blank your application must be able to handle the same base path that the gateway uses, otherwise you'll always receive a 404 status code.

For example: a request to _domain.com/service-a/123_ is proxied to "Service A" as _gw-service-a.azurewebsites.com/service-a/123_. How do you account for the additional path? You could map it in App Service directly using [virtual paths](https://learn.microsoft.com/en-us/azure/app-service/configure-common?tabs=portal#map-a-url-path-to-a-directory) or let your server handle it, e.g. in ASP.NET Core by calling `app.UsePathBase`.

To remove any gateway routes set the override to "/". See the [docs](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-http-settings#override-backend-path) for more information.

**Override with new host name** ensures that the App Service is able to accept a request in the first place. Since we cannot bind the same custom domain to multiple App Services, they must keep their _\*.azurewebsites.net_ (or different custom) domains. Any request with a different domain will be rejected. Therefore, the gateway must rewrite the domain.

> Make sure you are aware of the [limitations](https://learn.microsoft.com/en-us/azure/architecture/best-practices/host-name-preservation) that come with overriding the host name. Don't use path based routes if these limitations affect your application.

**Pick host name from backend target** simply specifies in which domain the gateway should rewrite it. Most of the time, that'll be the very host name that is specified in the backend pool.

##### Rules -> Add routing rule

Back in the rule form, scroll down to to the section "Path based routing".
Here you have to add a path based rule for each App Service:

- path "/service-a\*" routes to backend pool for App Service "A"
- path "/service-b\*" routes to backend pool for App Service "B"

The backend target we set before automatically acts as the default route, so you don't have to add it as a path here.
If you don't want to handle requests without path, simply create an empty backend pool. Application Gateway will respond with HTTP status code 502 in this case.

![Rule - all paths](/img/appservice-appgw/gw-add-https-rule-paths.jpeg)

##### Redirection to HTTPS

If you want to redirect traffic from HTTP to HTTPS, add another rule with HTTP as protocol. In the section for backend targets, select "Redirection" as target type and set the listener that you created for HTTPS as target listener.

#### Final steps

Your "Configuration" tab should now like this:

![Rule](/img/appservice-appgw/gw-final.jpeg)

After you confirm the creation of the gateway, your final task is to add the public IP address of the gateway to the DNS entries for your domain.
