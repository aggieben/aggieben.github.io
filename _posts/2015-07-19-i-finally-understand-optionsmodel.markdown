---
layout: post
title: "I Finally Understand OptionsModel"
modified:
categories:
excerpt:
tags: [asp.net-5]
comments: true
image:
  feature:
date: 2015-07-19T13:21:00-05:00
---

# Overview

I've been on the ASP.NET vNext crazy train for a number of months now, and at this stage of project I think the best way to learn the new framework (or contribute) is to dig around in the code.  The [aspnet](https://github.com/aspnet) user has a number of repositories that are active.  [aspnet/Home](https://github.com/aspnet/Home) is the banner for the entire effort, with links to other repositories, some simple samples, instructions on how to get started, and so on. [aspnet/Mvc](https://github.com/aspnet/Mvc) is another one that almost anyone even tangentially involved in vNext knows about (and it's the most active one). But there are many others that aren't headlined often, such as [aspnet/Security](https://github.com/aspnet/Security), or [aspnet/UserSecrets](https://github.com/aspnet/UserSecrets).  One I noticed a while back is [aspnet/Options](https://github.com/aspnet/Options).  

The repository description:

> A framework for accessing and configuring POCO settings.

Obviously, this is for accessing and configuring POCO settings ;-)  Keep in mind that as of now, there is very little documentation or even samples demonstrating the usage or motivation of OptionsModel.

# TL;DR

What I eventually came to realize is that OptionsModel is like a little bridge
between configuration objects and dependency injection.  It's a set of classes
that minimize your pain in getting configuration into your DI container as one
or more type-safe POCO settings objects in the form of `IOptions<TOptions>`,
where `TOptions` is your POCO.  It provides a uniform (and contravariant!)
interface for injecting and keeping references to your options classes, and it
supports idempotent initialization as well as ordered application of your
settings.  In the end, you can use the extension method
`IServiceCollection.Configure<TOptions>(/* ...overloaded... */)` to wire up your
POCO, and then it can be requested from the DI container as
`IOptions<TOptions>`, and you don't need to worry about what instance you're
getting or configuring.  One of the overloads of `Configure` that is
particularly useful is `IServiceCollection.Configure<TOptions>(IConfiguration)`,
which will take a configuration source and try to bind the values in it to your
POCO (which has to match the names and structure of your configuration).  See
the very end of this post for my simple example of usage.

# Getting Into It

While this seems perfectly obvious now, it wasn't at first and I want to walk
through my discovery process, showing examples and walking through how things
are connected.  This will be code-heavy, and I've reformatted somewhat to fit
the blog design, so bear with me...

What I found confusing at first was how this was really any different from just
grabbing settings out of any of the supported configuration sources (Memory,
INI, Json, Xml) - see
[aspnet/Configuration](https://github.com/aspnet/Configuration) for that.  Then
I found this:

{% highlight csharp linenos %}
public class ConfigureFromConfigurationOptions<TOptions> : ConfigureOptions<TOptions>
{
    public ConfigureFromConfigurationOptions(IConfiguration config)
        : base(options => ConfigurationBinder.Bind(options, config)) {}
}
{% endhighlight %}

This is from
[ConfigureFromOptions.cs](https://github.com/aspnet/Options/blob/dev/src/Microsoft.Framework.OptionsModel/ConfigureFromConfigurationOptions.cs).
It shed a little light on my question - see the
`ConfigurationBinder.Bind(options, config)` part?  That seemed important.  So
where was it?  It _was_ in [aspnet/Options](https://github.com/aspnet/Options),
but has moved to [aspnet/Configuration](https://github.com/aspnet/Configuration)
(see [aspnet/Options#65](https://github.com/aspnet/Options/pull/65)).  So what
does `ConfigurationBinder.Bind` do, anyway?  Here's a snippet that should give
you the idea:

{% highlight csharp linenos %}
public static class ConfigurationBinder
{
    public static TModel Bind<TModel>(IConfiguration configuration)
    	where TModel : new()
    {
        var model = new TModel();
        Bind(model, configuration);
        return model;
    }

    public static void Bind(object model, IConfiguration configuration)
    {
        if (model == null) { return; }
        BindObjectProperties(model, configuration);
    }

    private static void BindObjectProperties(object obj, IConfiguration configuration)
    {
        foreach (var property in GetAllProperties(obj.GetType().GetTypeInfo()))
        {
            BindProperty(property, obj, configuration);
        }
    }

    /* ... */
}
{% endhighlight %}

What should be apparent is that the point of this is to bind values to properties in in instance of `TModel`, which is just the POCO settings class we're interested in.  Read further on in [the code](https://github.com/aspnet/Configuration/blob/dev/src/Microsoft.Framework.Configuration.Binder/ConfigurationBinder.cs) to see all the details.  

At this point, OptionsModel seemed even more pointless to me - why would I bother with `IOptions` and so forth when I could just wire up my settings POCOs directly to the DI container?

{% highlight csharp linenos %}
public class Startup
{
    private readonly IConfiguration _configuration;

    public Startup(IApplicationEnvironment env)
    {
        _configuration = new ConfigurationBuilder(env.ApplicationBasePath)
                .AddJsonFile("config.json")
                .Build();
    }

    public void ConfigureServices(IServiceCollection services)
    {
        var settings = ConfigurationBinder.Bind<AppSettings>(
            	_configuration.GetConfigurationSection("AppSettings"));
        services.AddInstance(settings);
    }

    public void Configure(/*...*/)
    {
    	/* ... */
    }
}
{% endhighlight %}

With this, I can inject an `AppSettings` class anywhere else in my application, no problem.

Well, you _can_ do that.  But this isn't what OptionsModel is really for.  The next stop was to have a look at how Mvc is configured in a web application.  Going back to `Startup.cs`, here's what an Mvc app might look like, in part:

{% highlight csharp linenos %}
public class Startup
{
	/* ... */

	public void ConfigureServices(IServicesCollection services)
    {
        services.AddMvc();
        services.ConfigureMvc(options => options.Filters.Add(new FormatFilterAttribute()));
    }

    public void Configure(IApplicationBuilder app)
    {
    	app.UseMvc(routes =>
            routes.MapRoute("default", "{controller}", new { controller = "Home" }));
    }
}
{% endhighlight %}

So what's happening in `IServiceCollection.AddMvc()`?  Well, it's an extension
method located in
[MvcServiceCollectionExtensions.cs](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc/MvcServiceCollectionExtensions.cs).
The very first thing that happens there is `services.AddMvcCore()`.  We can find
that over in
[MvcCoreServiceCollectionExtensions.cs](https://github.com/aspnet/Mvc/blob/48bfdceea6d243c5ec8d6e00f450f8fe7cce59f7/src/Microsoft.AspNet.Mvc.Core/DependencyInjection/MvcCoreServiceCollectionExtensions.cs).
You'll quickly find that `AddMvcCoreServices()` is called, and in that method,
you find this:

{% highlight csharp linenos %}
internal static void AddMvcCoreServices(IServiceCollection services)
{
	services.TryAddEnumerable(
    	ServiceDescriptor.Transient<IConfigureOptions<MvcOptions>,
        							MvcCoreMvcOptionsSetup>());
    /* ... */
}
{% endhighlight %}

Now it becomes necessary to know what `MvcCoreMvcOptionsSetup` is.  It can be found in [MvcCoreMvcOptionsSetup.cs](https://github.com/aspnet/Mvc/blob/48bfdceea6d243c5ec8d6e00f450f8fe7cce59f7/src/Microsoft.AspNet.Mvc.Core/MvcCoreMvcOptionsSetup.cs), and here's the gist of it:

{% highlight csharp linenos %}
public class MvcCoreMvcOptionsSetup : ConfigureOptions<MvcOptions>
{
	public MvcCoreMvcOptionsSetup() : base(ConfigureMvc)
    {
    	Order = DefaultOrder.DefaultFrameworkSortOrder;
    }

    public static void ConfigureMvc(MvcOptions options)
    {
    	options.ModelBinders.Add(new BinderTypeBasedModelBinder());
        // ...

        options.OutputFormatters.Add(new StringOutputFormatter());
        // ...

        // other Mvc setup
    }
}
{% endhighlight %}

As an aside, this is a really great way to see a bunch of Mvc extension points all in in one place.  Anyway, you see that `ConfigureMvc` is given to the [`ConfigureOptions`](https://github.com/aspnet/Options/blob/dev/src/Microsoft.Framework.OptionsModel/ConfigureOptions.cs) constructor:

{% highlight csharp linenos %}
public class ConfigureOptions<TOptions> : IConfigureOptions<TOptions>
{
    public ConfigureOptions(Action<TOptions> action)
    {
    	Action = action;
    }

    public Action<TOptions> Action { get; private set; }
    public string Name { get; set; } = "";
    public virtual int Order { get; set; }
    	= OptionsConstants.DefaultOrder;

    public virtual void Configure(TOptions options, string name = "")
    {
        if (string.IsNullOrEmpty(Name)
            || string.Equals(name, Name, StringComparison.OrdinalIgnoreCase))
        {
            Action.Invoke(options);
        }
    }
}
{% endhighlight %}

You can see here that the `Action<TOptions>` (where `TOptions` is `MvcOptions`
in the Mvc's case) is just set up to be invoked later.  What's going on here is
that `Action` is going to be used to set up an `TOptions` object.

But before we really get what's going on, there are two more questions to answer:
1.  How does `TOptions` ever get instantiated?
2.  How do we assign our own configuration to `TOptions`?

Well, the answer to the first lies in [OptionsServiceCollectionExtensions.cs](https://github.com/aspnet/Options/blob/dev/src/Microsoft.Framework.OptionsModel/OptionsServiceCollectionExtensions.cs) in `ConfigureOptions()` and `FindIConfigureOptions()`:

{% highlight csharp linenos %}
private static IEnumerable<Type> FindIConfigureOptions(Type type)
{
    var serviceTypes = type.GetTypeInfo()
        .ImplementedInterfaces
        .Where(t => t.GetTypeInfo().IsGenericType
                && t.GetGenericTypeDefinition() == typeof(IConfigureOptions<>));

    if (!serviceTypes.Any())
    {
        string error = "TODO: No IConfigureOptions<> found.";
        if (IsAction(type))
        {
            error += " did you mean Configure(Action<T>)";
        }
        throw new InvalidOperationException(error);
    }
    return serviceTypes;
}

public static IServiceCollection ConfigureOptions(
    this IServiceCollection services,
    Type configureType)
{
    var serviceTypes = FindIConfigureOptions(configureType);
    foreach (var serviceType in serviceTypes)
    {
        services.AddTransient(serviceType, configureType);
    }
    return services;
}
{% endhighlight %}

Reflection is used to get the `IConfigureOptions<>` types configured in the type
being configured (I think there would typically be only one, although it's
interesting to note that `IFindConfigureOptions` seems to support multiple
implementations in a single type...), and then the configured type is added as a
transient mapping for each service type (again, this would typically be 1-to-1,
I would think).  The answer to question #1 is that the DI container instantiates
the `TOptions` type using a default constructor (that's what happens when that
form of `.AddTransient` is used).

As for #2, let's go back to Mvc.  Look in the sample `Startup.cs` snippet above,
where you see an invocation of `IServiceCollection.ConfigureMvc(options => /* ... */)`.
Go back to
[MvcCoreServiceCollectionExtensions.cs](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc.Core/DependencyInjection/MvcCoreServiceCollectionExtensions.cs)
to see what this does:

{% highlight csharp linenos %}
public static void ConfigureMvc(this IServiceCollection services,
                                Action<MvcOptions> setupAction)
{
	services.Configure(setupAction);
}
{% endhighlight %}

Now back to [OptionsServiceCollectionExtensions.cs](https://github.com/aspnet/Options/blob/dev/src/Microsoft.Framework.OptionsModel/OptionsServiceCollectionExtensions.cs):

{% highlight csharp linenos %}
public static IServiceCollection Configure<TOptions>(
    this IServiceCollection services,
    Action<TOptions> setupAction,
    int order = OptionsConstants.DefaultOrder,
    string optionsName = "")
{
	services.ConfigureOptions(new ConfigureOptions<TOptions>(setupAction)
    {
    	Name = optionsName,
        Order = order
    });
    return services
}
{% endhighlight %}

Now, this goes back up to the snippet above where the configured types get added
as transients (in this case, `ConfigureOptions<TOptions>`, which we've already
looked at.

Now we've answered half of #2: hopefully you can see now how your own
configuration (in the form of an `Action<TOptions>`) is set up.  What we haven't
seen yet is how the configurations are applied.  For that, let's go back to a
high level and drill down from another angle.  How do you request an instance of
your `TOptions` class?  In a constructor like this:

{% highlight csharp linenos %}
public class SomeController
{
	private readonly AppSettings _settings;
	public SomeController(IOptions<AppSettings> settings)
    {
    	_settings = settings.Options;
    }
	// ...
}
{% endhighlight %}

When we request an `IOptions<TOptions>` from the container, how is it resolved?
Look back at the top of [OptionsServiceCollectionExtensions.cs](http://):

{% highlight csharp linenos %}
public static IServiceCollection AddOptions(this IServiceCollection services)
{
    services.TryAdd(
        ServiceDescriptor.Singleton(typeof(IOptions<>),
                                    typeof(OptionsManager<>)));
    return services;
}
{% endhighlight %}

This looks a bit odd, but it's a neat trick taking advantage of the language
feature called "unbound generics", which are only useful for manipulation of
types.  Given an `OptionsManager<>`, one can do something like this:

{% highlight csharp linenos %}
Type unboundOptionsMgr = typeof(OptionsManager<>);
Type boundOptionsMgr = unboundOptionsMgr.MakeGenericType(typeof(AppSettings));
{% endhighlight %}

Once you have a `boundOptionsMgr`, it can be instantiated.  There's a very nice [explanation of this on StackOverflow](http://stackoverflow.com/a/2173115/3279).  

Anyway, the DI container will resolve requests for a closed generic type (i.e.,
`IOptions<TOptions>`) by matching it to the registration for the unbound generic
type `IOptions<>` and then closing the registered unbound generic type
`OptionsManager<>` using the `TOptions` from the request.  The resulting closed
generic type (`OptionsManager<TOptions>`) will be used to resolve requests for
`IOptions<TOptions>` (ref [DI
Notes](https://github.com/aspnet/DependencyInjection/wiki/DI-Notes#isomething-as-something---open-generic-services)).

Now, the last class we need to walk through, [`OptionsManager<TOptions>`](https://github.com/aspnet/Options/blob/dev/src/Microsoft.Framework.OptionsModel/OptionsManager.cs):

{% highlight csharp linenos %}
public class OptionsManager<TOptions> : IOptions<TOptions>
	where TOptions : class,new()
{
    private object _mapLock = new object();
    private Dictionary<string, TOptions> _namedOptions =
    	new Dictionary<string, TOptions>(StringComparer.OrdinalIgnoreCase);
    private IEnumerable<IConfigureOptions<TOptions>> _setups;

    public OptionsManager(IEnumerable<IConfigureOptions<TOptions>> setups)
    {
    	_setups = setups;
    }

    public virtual TOptions GetNamedOptions(string name)
    {
        if (!_namedOptions.ContainsKey(name))
        {
            lock (_mapLock)
            {
                if (!_namedOptions.ContainsKey(name))
                {
                    _namedOptions[name] = Configure(name);
                }
            }
        }
        return _namedOptions[name];
    }

    public virtual TOptions Configure(string optionsName = "")
    {
        return _setups == null
            ? new TOptions()
            : _setups
            	.OrderBy(setup => setup.Order)
                  .Aggregate(new TOptions(),
                              (options, setup) =>
                              {
                                  setup.Configure(options, optionsName);
                                  return options;
                              });
    }

    public virtual TOptions Options
    {
    	get { return GetNamedOptions(""); }
    }
}
{% endhighlight %}

Notice in the OptionsManager constructor that an
`IEnumerable<IConfigureOptions<TOptions>>` is required.  That means that when
the container resolves the closed generic type `OptionsManager<TOptions>`, it
will also have to resolve a list of `IConfigureOptions<TOptions>`.  Those are
the `ConfigureOptions<TOptions>` registered up above (go ahead and scroll back
to remind yourself if you need to).  The container will resolve the constructor
parameter as an array using all the registered types that match (this is really
nifty, IMO).  Now, finally, the last dot to connect: when we ask for our options
(`options.Options`), look what happens.  `GetNamedOptions("")` gets called, and
then if it's the first time, `Configure("")` will be called.  In configure, you
can see how the `IConfigureOptions` are put into proper order, and then executed
(`setup.Configure(options, optionsName)`), and then you get back a fully
constructed `TOptions`. SIMPLE, RIGHT??

# Climbing back out

Now that we've walked through all the machinery that makes this happen, and we
have the view of it from the inside out, what do you actually _do_ with it?
Hopefully knowing the inner workings should make that fairly evident: you don't
manage your own options lifetimes, you let OptionsModel do the work for you,
particularly if you have any requirements for idempotent options setup, or
stacked setups that may come from disparate sources (as would be the case in a
framework like Mvc).  Even if all I want to do is manage a POCO or two that come
from configuration, OptionsModel makes that really very simple (it has extension
method overloads to handle `IConfiguration`, specifically).  My new `Startup.cs`
looks like this:

{% highlight csharp linenos %}
public class Startup
{
    private readonly IConfiguration _configuration;

    public Startup(IApplicationEnvironment env)
    {
        _configuration =
        	new ConfigurationBuilder(env.ApplicationBasePath)
        	.AddJsonFile("config.json")
            .Build();
    }

    public void ConfigureServices(IServiceCollection services)
    {
    	services.Configure<AppSettings>(
        	_configuration.GetConfigurationSection("AppSettings"));
    }

    public void Configure(IApplicationBuilder app)
    {
    	// ...
    }
}
{% endhighlight %}

When I need access to options, I just ask for it in my constructors:

{% highlight csharp linenos %}
public class SomeController
{
	private readonly AppSettings _settings;
	public SomeController(IOptions<AppSettings> settings)
    {
    	_settings = settings.Options
    }
    // ...
}
{% endhighlight %}

# Conclusion

Pat yourself on the back for reading all this.
