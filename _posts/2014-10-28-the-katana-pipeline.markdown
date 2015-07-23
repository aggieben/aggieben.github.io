---
layout: post
title: "The Katana Pipeline"
modified:
categories:
excerpt:
tags: [asp.net-5]
image:
  feature:
date: 2014-10-28T00:00:00-05:00
---

Microsoft has undertaken a major effort to modernize its primary web framework, ASP.NET.  Unfortunately, the way those efforts have been communicated have been less than ideal. There have been a lot of different names of things: [Project K](http://www.zdnet.com/microsoft-shows-off-the-next-release-of-asp-net-7000029335/), [Helios](http://blogs.msdn.com/b/webdev/archive/2014/02/18/introducing-asp-net-project-helios.aspx), the [K Runtime](https://github.com/aspnet/KRuntime), [OWIN](http://owin.org/), [Katana](https://katanaproject.codeplex.com/), and  [ASP.NET vNext](http://www.asp.net/vnext).  Nevermind all the "I heard..." kind of hearsay.  Even people who can understand all this get confused.  *I'm* probably confused about some part of it.  What I can do here, though, is work through some of the key parts and try to arrive at some clarity.

#OWIN

First, let's review OWIN, or "Open Web Server Interface for .NET".  OWIN is *only* a spec.  It is not a software implementation; it is a document (or set of documents) that describe an interface for .NET servers and .NET web applications. The figure below should give you an idea of how the main pieces of OWIN fit together.

![Imgur](http://i.imgur.com/IhvMo9Y.png)

Note that IIS is both a server and a host (see the [specification](http://owin.org/spec/spec/owin-1.0.0.html#Definition) for more detailed information about exactly what each of these pieces are).  The Middleware layer represents a request-processing pipeline, composed of objects that implement the middleware interfaces.

#Katana

Microsoft's currently-released implementation of the Server and Middleware parts of this stack is called "Katana".  [The Katana project](http://www.asp.net/aspnet/overview/owin-and-katana/an-overview-of-project-katana) not only provides middleware and server components, but also  supporting artifacts, such as adapter layers that convert IIS semantics to OWIN semantics. Today's ASP.NET is an implementation of the Web Framework layer.

Strictly speaking, OWIN doesn't prescribe exactly how requests must be processed, just that the request processing must
be modeled by an application delegate with the following type:

{% highlight csharp linenos %}
using AppFunc = Func<IDictionary<string, object>, Task>;
{% endhighlight %}

The application must either complete the Task returned by this delegate or throw an exception.  Katana implements this application delegate by configuring (either in code or in the `web.config`) a set of middleware components during the OWIN startup.  For example:

{% highlight csharp linenos %}
public partial class Startup
{
	public void Configuration(IAppBuilder app)
    {
    	app.UseExternalSignInCookie(
        	DefaultAuthenticationTypes.ExternalCookie);
        app.UseGoogleAuthentication(
        	new GoogleOAuth2AuthenticationOptions
            {
                ClientId = "clientId",
                ClientSecret = "clientSecret",
            });
    }
}
{% endhighlight %}

In the sample code above, the Katana pipeline is configured to include middleware to authenticate users using Google OAuth2 authentication.  Once the `Configuration` method is complete, Katana takes the map of configured middleware and composes it into the `AppFunc` delegate defined above by chaining them together, so that each middleware calls the next one (or throws an exception) in the pipeline and then finally returns a `Task` asynchronously.

.NET lends itself to composing this application delegate of reusable interface implementations through the use of lambdas.

![OWIN Pipeline chart](http://i.imgur.com/smdVtgP.png)

A very natural expression of this kind of composition is a pipline as visualized at a very high level in the figure above.  Incoming client requests are handled by the Host and Server implementations (again, with IIS there isn't a meaningful distinction), and then a chain of Middleware components are given an opportunity to do additional processing before the request is handled by the application.

What is passed between each part of the pipeline is represented by an object with the `IDictionary<string, object>` interface.  The spec describes this quite succinctly:

> The Environment dictionary stores information about the request, the response, and any relevant server state.  The server is responsible for providing body streams and header collections for both the request and response in the initial call.  The application then populates the appropriate fields with response data, writes the response body, and returns when done. [[§3.2](http://owin.org/html/owin.html#32-environment)]

Middleware is considered to be part of the Application in this context.


##IIS Request Processing
The Katana pipeline sounds great, but how does it interact with the existing IIS/ASP.NET architecture?  This is illustrated in the diagram below.

![IIS pipeline](http://i.imgur.com/Mb5eq7I.png)

ASP.NET configures a routing module to process the `PostResolveRequestCache` and `PostMapRequestHandler` events and writes information about matching handlers to the context.  When the `Execute Handler` stage is processed, matching handlers are executed.  Katana provides an `IHttpHandler` implementation that translates a `System.Web` request into an OWIN context: [`OwinHttpHandler`](http://katanaproject.codeplex.com/SourceControl/latest#src/Microsoft.Owin.Host.SystemWeb/OwinHttpHandler.cs).

Another included host is the `HttpListener` host, which is exactly what it sounds like: an OWIN-compliant host that doesn't rely on IIS, but can receive incoming requests from any source.

A host that isn't included in Katana, but was [published separately](https://www.nuget.org/packages/Microsoft.Owin.Host.IIS/), is an IIS host without a dependency on System.Web.  This host used to be called [Project Helios](http://blogs.msdn.com/b/webdev/archive/2014/02/18/introducing-asp-net-project-helios.aspx) and leverages the capabilities of IIS, but doesn't involve System.Web in request processing.  The project is now published with ASP.NET vNext on myget.org as [Microsoft.AspNet.Server.IIS](https://www.myget.org/gallery/aspnetvnext).


##ASP.NET vNext

The next major release of ASP.NET vNext takes all this and makes it much more apparent [where all this is going](https://katanaproject.codeplex.com/wikipage?title=roadmap).  vNext takes Katana and folds it into the ASP.NET framework proper.  MVC and WebAPI are unified on this pipeline, and the project infrastructure has been rebuilt around a simpler, more readable JSON format.  Perhaps most importantly, vNext relieves applications of the traditional ASP.NET dependency on System.Web, which in turn has a dependency on IIS.  By implementing the OWIN specification, ASP.NET will be runnable with any host that likewise implements OWIN - which allows for a wide range of application architectures.  You can build an application that gets raw requests and writes diretly to the output, without pretty much anything in between.  You can still write System.Web applications (thanks to the Katana-cum-vNext System.Web-based host implementation), or you can write an application that takes advantage of the highly-configurable OWIN pipeline with pluggable middleware, à la Node.js - and it doesn't even have to be on Windows.

ASP.NET vNext "alpha4" was recently released along with [Visual Studio "14" CTP4](http://www.visualstudio.com/en-us/downloads/visual-studio-14-ctp-vs.aspx), which now supports Nuget packages and has stabilized the names of the runtimes (hopefully), so it is much more "ready" than it was even just a few weeks prior.

Be sure and check out the latest vNext developments at https://github.com/aspnet/Home.

---
Recommended resources:

http://herdingcode.com/herding-code-164-owin-and-katana-with-louis-dejardin/
https://support.microsoft.com/kb/2967191?wa=wsignin1.0
http://owin.org/html/owin.html
http://blogs.msdn.com/b/webdev/archive/2014/02/18/introducing-asp-net-project-helios.aspx
http://www.asp.net/aspnet/overview/owin-and-katana/an-overview-of-project-katana
https://katanaproject.codeplex.com/wikipage?title=roadmap
