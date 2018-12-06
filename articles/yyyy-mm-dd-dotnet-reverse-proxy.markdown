---
layout: post
title: "Building a Reverse Proxy in .NET Core"
description: "Create a reverse proxy in C# to meet your customized needs"
date: yyyy-mm-dd h:mm
category: Technical guide, .NET Core, C#
banner:
  text: "Auth0 makes it easy to add authentication to your Angular application."
author:
  name: "Andrea Chiarelli"
  url: "https://twitter.com/andychiare"
  mail: "andrea.chiarelli.ac@gmail.com"
  avatar: "https://cdn.auth0.com/blog/guest-author/andrea-chiarelli.jpg"
design:
  image: https://cdn.auth0.com/blog/angular/logo3.png
  bg_color: "#012C6C"
tags:
- .net core
- c#
- reverse proxy
- http
related:
- 2017-11-09-sqlalchemy-orm-tutorial-for-python-developers
- 2017-09-28-developing-restful-apis-with-python-and-flask

---

**TL;DR:** The article will show how to implement a *reverse proxy* in C# within the *.NET Core* environment to overcome specific needs that you could hardly solve with an out-of-the-box software. You can find the code of the final project on this [GitHub repository](https://github.com/andychiare/netcore2-reverse-proxy).

---

## Using a Reverse Proxy

Among the various elements of a network infrastructure, such as DNS servers, firewalls, proxies and similar, reverse proxies gained a lot of popularity not only among IT people but also among developers. This is mainly due to the growth in popularity of microservice architectures and to advanced integration needs between technical partners.

### What is a Reverse Proxy

A standard [proxy server](https://en.wikipedia.org/wiki/Proxy_server) acts as an intermediary between a client and a server in order to perform processing like caching, traffic monitoring, resource access control, etc. The relevant thing is that the client requests to contact a specific server and the proxy, after evaluating how to satisfy its request, contacts the target server on behalf of the client.

A [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) is a special type of proxy server that hides the target server to the client. The client requests a resource to the proxy server which retrieves it from another server and provides it to the client. In this case, the client has no idea that the resource comes from another server.

### When Using a Reverse Proxy

A reverse proxy can be used in different contexts:

- *load balancing*

  Maybe this is one of the most familiar uses of a reverse proxy. It can distribute the requests load among a set of identical servers accordingly to some specific algorithm providing support for scalability and availability of a system.

- *URL rewriting*

  Having meaningful URLs is crucial from an SEO standpoint. If you cannot change the URLs of your website, you can hide them behind a reverse proxy that exposes to users and web crawlers a more attractive version.

- *static content serving*

  Many reverse proxy servers may be configured to act as web servers. This allows you to use them for providing static content, such as HTML pages, JavaScript scripts, images, and other files while forwarding requests for dynamic content to dedicated servers. This is a sort of load balancing based on the content type.

- *API gateway*

  In a system with a microservice architecture, you have multiple servers offering different services through their APIs. You can use a reverse proxy to expose a single entry point for the combination of the server's APIs

- *multiple website combining*

  This is pretty similar to the API gateway context. In this case, you can have a single entry point for multiple websites, possibly with a centralized homepage.

### Why Implementing a Custom Reverse Proxy

One of the most popular reverse proxies is [nginx](http://nginx.org/). Of course, you can use other tools, like [Pound](http://www.apsis.ch/pound/) or [Squid](http://www.squid-cache.org/), or you can also configure the [Apache Web server to act as a reverse proxy](http://httpd.apache.org/docs/current/howto/reverse_proxy.html). These tools offer a lot of configuration options that allow you to set up your system in most common scenarios. Some of them also provide the ability to extend their functionality with scripting languages, such as, for example, the [Lua](https://www.lua.org/) scripting language in nginx. This capability allows you to face some processing needs that the simple server configuration doesn't provide: HTTP header manipulation, conditional request forwarding, simple content transformation.

However, you may find scenarios where even an integrated scripting language is not enough for your needs due to the complexity of the scenario itself or because the scripts become hard to maintain. Consider, for example, a scenario where you need to expose a remote Web application within the current domain and need to prepare the HTTP requests by injecting data from a database and manipulate the responses to integrate them in the current environment. Or a scenario where you need to apply complex custom rules to analyze the HTTP traffic.

If you are in such a situation you may need to build your own custom reverse proxy.

## Implementing a Reverse Proxy in C#

Implementing the core of your own reverse proxy is not so hard as it may seem. Of course, you will be unlikely to create a reverse proxy with all the options that nginx or other similar tools can provide. However, you could focus on your specific goal in order to solve it at your best without resorting to complex configurations and scripts.

In the remainder of the article, you will build a simple reverse proxy in C# that will allow you to integrate in your website a Google form. The form is [publicly available without authentication](https://docs.google.com/forms/d/e/1FAIpQLSdJwmxHIl_OCh-CI1J68G1EVSr9hKaYFLh3dHh8TLnxjxCJWw/viewform?hl=en) and allows to register to receive a T-shirt. When integrated into your Web application through the reverse proxy, it will be prefilled with some personal data of the current user. Of course, the implementation of the Web application will be kept simple in order to focus on the challenges related to the reverse proxy. Let's start coding!

### Setting up the Project

You can create a new .NET Core project by using Visual Studio or via command line.

By using Visual Studio, you choose the ASP.NET Core template as shown in the following picture:

![](images/visual-studio-new-project.png)

In the next screen, you select the *Empty* project template, as shown below:

![](images/visual-studio-empty-project.png)

You can create the same project template by typing the following command in a console window:

```shell
dotnet new web -n ReverseProxyApplication
```

You should get the following screen in a few seconds:

![](images/command-line-new-project.png)

Regardless of your choice, you will get the minimal project files for an empty ASP.NET application in the folder you have specified.

### Adding the Reverse Proxy Middleware

Recalling the definition of a reverse proxy, you need to intercept some HTTP requests and redirect them to another server without the client knowing it. In the .NET Core infrastructure, you can obtain this by implementing a middleware, that is a component that you can inject in the HTTP pipeline in order to handle requests and responses.

So, add a new file to the project named `ReverseProxyMiddleware.cs` with the following content:

```c#
// ReverseProxyApplication/ReverseProxyMiddleware.cs
using Microsoft.AspNetCore.Http;
using System;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;

namespace ReverseProxyApplication
{
    public class ReverseProxyMiddleware
    {
        private static readonly HttpClient _httpClient = new HttpClient();
        private readonly RequestDelegate _nextMiddleware;

        public ReverseProxyMiddleware(RequestDelegate nextMiddleware)
        {
            _nextMiddleware = nextMiddleware;
        }

        public async Task Invoke(HttpContext context)
        {
            var targetUri = BuildTargetUri(context.Request);

            if (targetUri != null)
            {
                var targetRequestMessage = CreateTargetMessage(context, targetUri);

                using (var responseMessage = await _httpClient.SendAsync(targetRequestMessage, HttpCompletionOption.ResponseHeadersRead, context.RequestAborted))
                {
                    context.Response.StatusCode = (int)responseMessage.StatusCode;
                    CopyFromTargetResponseHeaders(context, responseMessage);
                    await responseMessage.Content.CopyToAsync(context.Response.Body);
                }
                return;
            }
            await _nextMiddleware(context);
        }

        private HttpRequestMessage CreateTargetMessage(HttpContext context, Uri targetUri)
        {
            var requestMessage = new HttpRequestMessage();
            CopyFromOriginalRequestContentAndHeaders(context, requestMessage);

            requestMessage.RequestUri = targetUri;
            requestMessage.Headers.Host = targetUri.Host;
            requestMessage.Method = GetMethod(context.Request.Method);

            return requestMessage;
        }

        private void CopyFromOriginalRequestContentAndHeaders(HttpContext context, HttpRequestMessage requestMessage)
        {
            var requestMethod = context.Request.Method;

            if (!HttpMethods.IsGet(requestMethod) &&
                !HttpMethods.IsHead(requestMethod) &&
                !HttpMethods.IsDelete(requestMethod) &&
                !HttpMethods.IsTrace(requestMethod))
            {
                var streamContent = new StreamContent(context.Request.Body);
                requestMessage.Content = streamContent;
            }

            foreach (var header in context.Request.Headers)
            {
                requestMessage.Content?.Headers.TryAddWithoutValidation(header.Key, header.Value.ToArray());
            }
        }

        private void CopyFromTargetResponseHeaders(HttpContext context, HttpResponseMessage responseMessage)
        {
            foreach (var header in responseMessage.Headers)
            {
                context.Response.Headers[header.Key] = header.Value.ToArray();
            }

            foreach (var header in responseMessage.Content.Headers)
            {
                context.Response.Headers[header.Key] = header.Value.ToArray();
            }
            context.Response.Headers.Remove("transfer-encoding");
        }
        private static HttpMethod GetMethod(string method)
        {
            if (HttpMethods.IsDelete(method)) return HttpMethod.Delete;
            if (HttpMethods.IsGet(method)) return HttpMethod.Get;
            if (HttpMethods.IsHead(method)) return HttpMethod.Head;
            if (HttpMethods.IsOptions(method)) return HttpMethod.Options;
            if (HttpMethods.IsPost(method)) return HttpMethod.Post;
            if (HttpMethods.IsPut(method)) return HttpMethod.Put;
            if (HttpMethods.IsTrace(method)) return HttpMethod.Trace;
            return new HttpMethod(method);
        }

        private Uri BuildTargetUri(HttpRequest request)
        {
            Uri targetUri = null;

            if (request.Path.StartsWithSegments("/googleforms", out var remainingPath))
            {
                targetUri = new Uri("https://docs.google.com/forms" + remainingPath);
            }

            return targetUri;
        }
    }
}
```

Here you have defined a `ReverseProxyMiddleware` class with two private properties:

```c#
private static readonly HttpClient _httpClient = new HttpClient();
private readonly RequestDelegate _nextMiddleware;
```

The `_httpClient` property defines the HTTP client you will use to pass requests to the target server, while the `_nextMiddleware` property represents any subsequent middleware in the ASP.NET HTTP pipeline. You initialize the `_nextMiddleware` property in the class constructor as follows:

```c#
public ReverseProxyMiddleware(RequestDelegate nextMiddleware)
{
  _nextMiddleware = nextMiddleware;
}
```

Most of the work is done by the `Invoke()` method, as you can see in the following snippet of code:

```c#
public async Task Invoke(HttpContext context)
{
  var targetUri = BuildTargetUri(context.Request);

  if (targetUri != null)
  {
    var targetRequestMessage = CreateTargetMessage(context, targetUri);

    using (var responseMessage = await _httpClient.SendAsync(targetRequestMessage, HttpCompletionOption.ResponseHeadersRead, context.RequestAborted))
    {
      context.Response.StatusCode = (int)responseMessage.StatusCode;
      CopyFromTargetResponseHeaders(context, responseMessage);
      await responseMessage.Content.CopyToAsync(context.Response.Body);
    }
    return;
  }
  await _nextMiddleware(context);
}
```

It tries to build the target Uri, that is the address of the target server, starting from the current HTTP context. If a target Uri has been returned by the `BuildTargetUri()` method, it means that the original HTTP request should be forwarded to the target server, so the original request must be processed. Otherwise, the request is not processed by the current middleware and it is passed to the next middleware in the pipeline.

When the request needs to be processed, it builds a message for the target server through the `CreateTargetMessage()` method and sends it by using the `_httpClient` private property. Then, the response received from the target server is entirely copied into the response to be provided to the client.

In summary, this code describes the standard workflow of a reverse proxy. You can find a few methods in the class supporting the copy of the content and the headers of the HTTP request and response, but the more interesting code is the one that builds the target Uri:

```c#
private Uri BuildTargetUri(HttpRequest request)
{
  Uri targetUri = null;

  if (request.Path.StartsWithSegments("/googleforms", out var remainingPath))
  {
    targetUri = new Uri("https://docs.google.com/forms" + remainingPath);
  }

  return targetUri;
}
```

As you can see, it looks for requests whose path starts with the `/googleforms` string and returns a new Uri by replacing that string with `https://docs.google.com/forms`, that is the base Uri of the  Google Forms service. If the `/googleforms` prefix is not found, `null` will be returned.

### Using the Reverse Proxy Middleware

Now you are ready to use the reverse proxy middleware in the ASP.NET application. Open the `Startup.cs` file and modify the `Configure()` method as shown in the following example:

```C#
// ReverseProxyApplication/Startup.cs
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
  if (env.IsDevelopment())
  {
    app.UseDeveloperExceptionPage();
  }

  app.UseMiddleware<ReverseProxyMiddleware>();

  app.Run(async (context) =>
  {
    await context.Response.WriteAsync("<a href='/googleforms/d/e/1FAIpQLSdJwmxHIl_OCh-CI1J68G1EVSr9hKaYFLh3dHh8TLnxjxCJWw/viewform?hl=en'>Register to receive a T-shirt</a>");
  });
}
```

You added the reverse proxy middleware to the HTTP pipeline through the `UseMiddleware()` method and changed the default content sent to the browser with a link pointing to the Google form via the internal Uri prefix `/googleforms`. After this change, launch the Web application by pressing the start button in Visual Studio or by typing the `dotnet run` command in a console window.

> The very first run from Visual Studio should display the following alert:
>
> ![](images/visual-studio-certificate-alert.png)
>
> This happens because by default Visual Studio configures the ASP.NET application to use HTTPS. So, the at the first run, an SSL certificate will be installed on your system. You should accept this first alert and the subsequent alert asking confirmation about the certificate installation.

When the ASP.NET application is running, you can point your browser to `localhost:5001` getting the following page:

![](images/home-page.png)

This page shows just the link you set in the `Configure()` method. By clicking this link, you should get a page like the following:

![](images/register-for-t-shirt.png)

Awesome! It is working. Notice that the URL in the address bar remains inside your application domain.

However, if you try to select the size of the T-shirt or to add a comment, you feel that something is wrong. Actually, if you open the developer tools of your browser, you get a lot of error messages:

![](images/cors-error-messages.png)

What is happening? Why there are so many errors?

Your reverse proxy passed the resources coming from the `https://docs.google.com/forms` domain to your browser, but some of these resources are JavaScript script containing Ajax requests to another domain: `https://www.gstatic.com`. This is correctly interpreted by your browser as an attempt to breach the [same origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy). 

### Processing the Response of the Target Server

To avoid the error due to the *the same origin policy* violation, you should make the scripts performing the Ajax requests to be loaded through the reverse proxy. In addition, you should also ensure that even the Ajax requests will be submitted to the target server through the reverse proxy. This ensures Ajax calls to comply with the browsers' same origin policy, while the reverse proxy directs the HTTP requests to the correct servers.

Checking the network requests, you find that the scripts are loaded from the domains `https://www.google.com` and `https://www.gstatic.com`. So, we need to capture the references to these domains inside the HTML and JavaScript content coming from the target server and replace it with a prefix in our domain, say `/google` and `/googlestatic`. In order to get this result, you need to change the `Invoke()` method of the `ReverseProxyMiddleware` class. In particular, you need to replace  its body with the following:

```c#
// ReverseProxyApplication/ReverseProxyMiddleware.cs
public async Task Invoke(HttpContext context)
{
  var targetUri = BuildTargetUri(context.Request);

  if (targetUri != null)
  {
    var targetRequestMessage = CreateTargetMessage(context, targetUri);

    using (var responseMessage = await _httpClient.SendAsync(targetRequestMessage, HttpCompletionOption.ResponseHeadersRead, context.RequestAborted))
    {
      context.Response.StatusCode = (int)responseMessage.StatusCode;
      CopyFromTargetResponseHeaders(context, responseMessage);
      await ProcessResponseContent(context, responseMessage);
    }
    return;
  }
  await _nextMiddleware(context);
}
```

The only difference from the previous version is the call to the `ProcessResponseContent()` instead of the statement `await responseMessage.Content.CopyToAsync(context.Response.Body)`.

This new private method is responsible for processing the response coming from the target server. Its implementation is as follows:

```c#
// ReverseProxyApplication/ReverseProxyMiddleware.cs
private async Task ProcessResponseContent(HttpContext context, HttpResponseMessage responseMessage)
{
  var content = await responseMessage.Content.ReadAsByteArrayAsync();

  if (IsContentOfType(responseMessage, "text/html") || 
      IsContentOfType(responseMessage, "text/javascript"))
  {
    var stringContent = Encoding.UTF8.GetString(content);
    var newContent = stringContent.Replace("https://www.google.com", "/google")
        .Replace("https://www.gstatic.com", "/googlestatic")
        .Replace("https://docs.google.com/forms", "/googleforms");;
    await context.Response.WriteAsync(newContent, Encoding.UTF8);
  } else {
    await context.Response.Body.WriteAsync(content);
  }
}
```

As you can see, it extracts the content of the response from the target server and checks its content type. If the response body contains HTML markup or JavaScript code, then any occurrence of the domains detected above are replaced by the respective prefixes and the response is changed accordingly. Otherwise, the original content is forwarded to the client.

In the same class, you need to change also the `BuildTargetUri()` private method, so that these new path prefixes are processed accordingly. This is the new version of the method:

```c#
// ReverseProxyApplication/ReverseProxyMiddleware.cs
private Uri BuildTargetUri(HttpRequest request)
{
  Uri targetUri = null;
  PathString remainingPath;

  if (request.Path.StartsWithSegments("/googleforms", out remainingPath))
  {
    targetUri = new Uri("https://docs.google.com/forms" + remainingPath);
  }

  if (request.Path.StartsWithSegments("/google", out remainingPath))
  {
    targetUri = new Uri("https://www.google.com" + remainingPath);
  }

  if (request.Path.StartsWithSegments("/googlestatic", out remainingPath))
  {
    targetUri = new Uri(" https://www.gstatic.com" + remainingPath);
  }

  return targetUri;
}
```

Now you can run the application and this time you should be able to successfully register to receive a brand new T-shirt without leaving your application's domain.

### Processing the Original Request

As per initial specifications, you should display the registration form to the user with some pre-filled data, such as the user's name. The Google Forms application allows to prefill a form by passing the field name and its value in the query string. So, you can inspect the generated form markup and get the input element's name, as shown by the following picture:

![](images/input-value-inspection.png)

Having this information, you should simply append to the query string of original request Uri the fragment `entry.1884265043=John%20Doe`, assuming that *John Doe* is the actual name of the current user.

So, change the `CreateTargetMessage()` private method of the `ReverseProxyMiddleware` class as follows:

```c#
// ReverseProxyApplication/ReverseProxyMiddleware.cs
private HttpRequestMessage CreateTargetMessage(HttpContext context, Uri targetUri)
{
  var requestMessage = new HttpRequestMessage();
  
  CopyFromOriginalRequestContentAndHeaders(context, requestMessage);

  targetUri = new Uri(QueryHelpers.AddQueryString(targetUri.OriginalString, 
             new Dictionary<string, string>() { { "entry.1884265043", "John Doe" } }));

  requestMessage.RequestUri = targetUri;
  requestMessage.Headers.Host = targetUri.Host;
  requestMessage.Method = GetMethod(context.Request.Method);

  return requestMessage;
}
```

You added the `targetUri` re-definition statement, where you added the query string fragment. For simplicity, here you assigned a constant string value as the name of the current user. In a real case, you should retrieve the name of the user in a more complex way, from the session data, from a database or from another data source.

Now, you can run again the Web application and when you go to the second page of the form you will find the prefilled textbox, as shown in the following picture:

![](images/prefilled-form.png)



## Summary

After exploring what a reverse proxy is, when using it and why you could need to implement a custom one, you started setting up an ASP.NET Core application to learn how to implement it. You created a middleware acting as a reverse proxy: it captured specific requests and submitted them to the target servers. In the application example, you have implemented, the target servers were a few Google's servers. During the implementation, you explored how to process the target server's responses in order to avoid violations of *the same origin policy* of the browser and how to process the original requests in order to get a prefilled form.

You can find the final code of the project developed throughout this article on [this GitHub repository](https://github.com/andychiare/netcore2-reverse-proxy).

