---
layout: "post"
title: "Solutions For Async/Await In MVC Action Filters"
description: ''
category: null
tags: []
---

{% include JB/setup %}

Async/await has been available in .net for years but until the release of asp.net core there was no way to create a MVC `ActionFilter` that uses async/await properly. Since async was not supported by the framework there was no truly safe way to call async code from an `ActionFilter`. This has changed in asp.net core but if you are using ASP.Net 5 or below you're stuck.

Recently, I found a workaround to using an async `HttpModule` to load whatever data the `ActionFilter` will need. You could also do all the work of the `ActionFilter` in the `HttpModule` but I prefer to keep the filter because it ties more closely into the rest of the MVC pipeline. My example will demonstrate moving async code out of an `AuthorizationFilter` but the pattern will work with any `ActionFilter`.

#### ActionFilter to fix
This is an example authorization filter that does async work as part of the authorization of the request. Because attributes do not have async methods to override we're stuck calling `.Wait()` and `.Result` to synchronously execute the task. This code is [ripe for deadlocks](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html).

~~~
public class WebAuthorizationFilter : AuthorizeAttribute
{
  public override void OnAuthorization( AuthorizationContext filterContext )
  {
     if ( AllowAnonymous( filterContext ) )
     {
        return;
     }

     Task<bool> isAuthorizedTask = DoAsyncAuthorizationWork( filterContext.HttpContext );
     isAuthorizedTask.Wait();

     bool isAuthorized = isAuthorizedTask.Result;
     if ( !isAuthorized)
     {
        filterContext.Result = new UnauthorizedResult();
     }
  }
}
~~~

There are 2 classes necessary to add the module:

#### New HttpModule to handle async code
Any async code goes in this class. Call necessary methods then add state to `HttpContext.Items`.
~~~
public class WebAuthorizationAsyncModule : IHttpModule
{
    public void Init( HttpApplication context )
    {
       var authWrapper = new EventHandlerTaskAsyncHelper( AuthorizeRequestAsync );

       // Execute module early in pipeline during request authorization
       // To execute the module after the MVC route has been bound, use `context.AddOnPostAcquireRequestStateAsync` instead
       context.AddOnAuthorizeRequestAsync( authWrapper.BeginEventHandler, authWrapper.EndEventHandler );
    }

    private static async Task AuthorizeRequestAsync( object sender, EventArgs e )
    {
       HttpApplication httpApplication = (HttpApplication) sender;
       HttpContext context = httpApplication.Context;

       bool isAuthorized = await DoAsyncAuthorizationWork( context );

       // Store the result in HttpContext.Items for later access
       context.Items.Add( "IsAuthorized", isAuthorized );
    }
}
~~~

#### Module Registration Startup Class
This class registers the `HttpModule` created above with asp.net. You can also register in the web.config but I prefer to keep this kind of configuration in code.
~~~
public class PreApplicationStartCode
{
  public static void Start()
  {
    DynamicModuleUtility.RegisterModule( typeof( WebAuthorizationAsyncModule ) );
  }
}
~~~

The second code change required is to add the `PreApplicationStartCode` class to the startup classes registered with asp.net. To do this use the `PreApplicationStartMethod` attribute on your `HttpApplication` class in `Global.asax.cs`.
~~~
[assembly: PreApplicationStartMethod( typeof( Some.Code.PreApplicationStartCode ), "Start" )]
namespace Some.Code
 {
    public class WebApplication : HttpApplication
    {
      ...
~~~

#### Action Filter changes
This is the same authorization filter from above changed to read the authorization result from `Httpcontext.Items` instead of doing the work directly.
~~~
public class WebAuthorizationFilter : AuthorizeAttribute
{
  public override void OnAuthorization( AuthorizationContext filterContext )
  {
     if ( AllowAnonymous( filterContext ) )
     {
        return;
     }

     bool isAuthorized =  (bool) filterContext.HttpContext.Items["isAuthorized"];
     if ( !isAuthorized)
     {
        filterContext.Result = new UnauthorizedResult();
     }
  }
}
~~~

#### Next Steps
This example was deliberately simple and as such it executes for every request. To only execute the module for specific URLs you can inspect the `HttpContext.Request.Url` property. Or if you delay execution of the module until after the 'Acquire State' pipeline step in asp.net (`context.AddOnPostAcquireRequestStateAsync` in Module.Init) and access the MVC route values using `HttpContext.Request.RequestContext.RouteData.Values` you can only execute the module code for specific Controllers/Actions in MVC.
