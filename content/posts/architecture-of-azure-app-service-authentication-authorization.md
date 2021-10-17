---
title: "Architecture of Azure App Service Authentication / Authorization"
date: "2016-02-01"
categories: 
  - "easy-auth"
---

Authentication / Authorization (which I'll refer to as **Easy Auth** throughout this post) is a feature of Azure App Service that allows you to easily integrate a variety of auth capabilities into your web app or API. It's built directly into the platform and doesn't require any particular languages, SDKs, security expertise, or even any code to utilize. This is why we call it _easy_ - anybody can leverage it, even non-developers, with just a few clicks in the management portal.

In this post we will look at how Easy Auth is built and the various design tradeoffs involved. Be warned: We'll go into deep technical detail in this post. If instead you'd like to learn more about what this feature is at a high level and how to use it, take a look at [our overview topic in the Azure documentation](https://azure.microsoft.com/documentation/articles/app-service-authentication-overview/) and/or [this announcement](https://azure.microsoft.com/blog/announcing-app-service-authentication-authorization/) by my colleague [Matthew Henderson](https://twitter.com/mattchenderson). Otherwise, continue reading.

# Architecture

Easy Auth is implemented as a native [IIS module](http://www.iis.net/learn/get-started/introduction-to-iis/iis-modules-overview) that runs in the same sandbox as your application. When enabled, every HTTP request dispatched to the IIS worker process must first pass through this module before your application code has a chance to react.

![Easy Auth Runtime Architecture](/images/Easy-Auth-Architecture-1-1024x543.png)

As you can see, the module runs in the same sandbox as your application, but is still very much separate from your application. It's automatically injected when enabled via the management portal. No SDKs, specific languages, or any code changes are required. It is configured using environment variables, which are internally populated by the portal (or the Azure Resource Management REST APIs).

# Alternatives

The do-it-yourself way to accomplish this could have been to use include your favorite auth middleware directly into your application code. As a developer, this gives you a lot more control, but requires more work and requires you to know what you're doing (OAuth 2, OpenID Connect, cookies, encryption, HMACs, CSRF, replay-attacks, etc.), which can be scary if it's security-related. The reality is that some people don't want to have to think about these things. Why not have the platform take care of it for you?

Another logical way to achieve the same outcome could have been to leverage an authentication proxy which sits in front of your app. You may have even seen some Microsoft products which do this, such as [API Management](https://azure.microsoft.com/services/api-management/). However, it was decided that this approach is not ideal for the most common types of application backends.

# Platform Integration

The best part of incorporating the authentication layer directly into the platform is that it can integrate very nicely with the platform. This ultimately benefits you, the customer, in a variety of ways. A few examples:

- **Integrated Identity**: If you're writing a legacy ASP.NET 4.x app, you can access the identity and claims associated with the authenticated user using the [`ClientPrincipal.CurrentPrincipal`](https://msdn.microsoft.com/library/system.security.claims.claimsprincipal.current%28v=vs.110%29.aspx) API. Web API attributes such as [`[Authorize]`](https://msdn.microsoft.com/library/system.web.http.authorizeattribute%28v=vs.118%29.aspx) also work natively. Even PHP apps benefit because we automatically set the [`REMOTE_USER` server variable](http://php.net/manual/en/features.http-auth.php), allowing PHP apps to infer the identity of the user. This is _huge_ for easily enabling you to do things like creating corporate [MediaWiki](https://www.mediawiki.org/wiki/MediaWiki) apps (something we do internally at Microsoft).
- **Logging**: If you're using the [Application Logs](https://azure.microsoft.com/documentation/articles/web-sites-enable-diagnostic-log/) feature of App Service (which is an awesome feature, by the way), you'll notice that Easy Auth traces are included directly in your application traces. If you see an authentication error that you didn't expect, you can conveniently find all the details by looking in your existing application logs.
- **Error Tracing**: For more advanced debugging, you enable [Failed Request Tracing](http://blogs.msdn.com/b/benjaminperkins/archive/2013/08/01/enabling-failed-request-logging-on-a-windows-azure-web-site.aspx) in your application. Because Easy Auth is an IIS module that runs in-process with your app, you can see exactly what role it might have played in a failed request. Just looks for references to a module named `EasyAuthModule_32/64`. Auth middleware, on the other hand, does not have this advantage because IIS can't distinguish it from your application code.
- **File System Storage**: If you have the Easy Auth Token Store enabled, we can cache OAuth tokens directly on disk rather than needing to provision and manage a separate storage account. I'll talk more about the token store in a future post.
- **Configuration**: All Easy Auth configuration is surfaced to the application as app settings. This is useful because your application code can easily read these settings. Take a look at the Kudu environment page to see them for yourself - all Easy Auth settings are prefixed with `WEBSITE_AUTH_`. This also makes it easier for us to expose experimental features that you can turn on using app settings.
- **Updates**: New features and improvements can be added to your app without requiring you redeploy any part of your application.

There may be other benefits that I've missed, but I think you get the basic idea of why we decided on this integrated approach to authentication in App Service.

# Performance

In terms of performance, the goal of Easy Auth is to ensure that the overhead of this feature is non-noticeable and/or an improvement over alternative authentication solutions. This design is already an improvement over gateway-architectures simply because there is no additional network hop between clients and your application. The Easy Auth module runs in-process with the application host (w3wp.exe).

In terms of the code itself, Easy Auth is actually built using mostly managed code. The attentive reader may have noticed that I said this is a _native_ module earlier in this post, and it still is. However, the native layer is just a small shim between IIS and the core module implementation. What this means practically is that we can engineer an IIS module in .NET without taking a dependency on ASP.NET and System.Web. This is important because [traditional managed modules](http://www.iis.net/learn/develop/runtime-extensibility/developing-a-module-using-net) (or more specifically ASP.NET) can incur a [relatively heavy performance penalty](http://blogs.msdn.com/b/webdev/archive/2014/02/18/supplemental-to-asp-net-project-helios.aspx) which serves no benefit when hosting non-ASP.NET applications (PHP, Node.js, etc.).

# Wrapping Up

Hopefully this information is useful, interesting, or both. If you have general questions or if you're running into issues, I suggest posting on [StackOverflow](http://stackoverflow.com) (please tag your questions with **azure-app-service** or **azure-web-sites**) so that others can easily find, rate, and participate in the discussion. For the highest level of support, take a look at some of our [support plans](https://azure.microsoft.com/support/plans/). This is absolutely the best way to get our attention and prioritize any issues you may encounter.

Also, feel free to reach out to me directly on [Twitter](https://twitter.com/cgillum).
