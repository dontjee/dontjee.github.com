---
layout: "post"
title: "Full Framework WSFederation to OWIN Conversion"
description: ''
category: null
tags: []
---

If you have been using WSFederation in a .net web application for more than a year or two, chances are that it is configured using the `Microsoft.IdentityModel.Web` or `System.IdentityModel.Services` libraries. Two HTTP modules are added to the application, `WSFederationAuthenticationModule` and `SessionAuthenticationModule`, to handle the WSFederation protocol and configuration was done by inheriting those classes or configuring on application start via the web.config. However, in newer versions of asp.net using "middleware" is preferred by using [OWIN](https://docs.microsoft.com/en-us/aspnet/aspnet/overview/owin-and-katana/an-overview-of-project-katana#the-open-web-interface-for-net-owin) in both full framework applications and .net core. The purpose of OWIN is to abstract the underlying web server from the web application. HttpModules are tightly coupled to `System.Web` and therefore the IIS webserver. Using OWIN does require some configuration and setup changes which I will detail in this post.

### Basic OWIN setup
First, if you don't already have OWIN configured for your application install the `Microsoft.Owin` and `Microsoft.Owin.Host.SystemWeb` nuget packages. Then add a startup class like the one below to your application:

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.Owin;
using Owin;

[assembly: OwinStartup(typeof(OwinApp.Startup))]
namespace OwinApp
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
        }
    }
}
```

### Convert SessionAuthenticationModule into OWIN configuration
Once OWIN is installed, we can begin configuring WSFederation. Previously a `SessionAuthenticationModule` would have been customized to set up properties for the cookies that will store session information:

```csharp
public class CustomSessionAuthenticationModule : SessionAuthenticationModule
{
  protected override void InitializePropertiesFromConfiguration()
  {
      CookieHandler.RequireSsl = true;
      CookieHandler.Name = "FederatedAuthCookie";
  }
}
```

and configured as a HTTP module in the web.config:

```xml
<modules>
  <add name="SessionAuthenticationModule" type="MyApp.CustomSessionAuthenticationModule, MyApp" preCondition="managedHandler" />
</modules>
```

In the OWIN pipeline, we'll configure the cookie using `CookieAuthentication` classes and helper methods.

```csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        app.UseCookieAuthentication( new CookieAuthenticationOptions
        {
          // converted from the CookieHandler.Name = "FederatedAuthCookie"; line in SessionAuthenticationModule
          CookieName = "FederatedAuthCookie",
          // converted from the CookieHandler.RequireSsl = true; line in SessionAuthenticationModule
          CookieSecure = CookieSecureOption.Always
        } );
    }
}
```

### Convert WSFederationAuthenticationModule into OWIN configuration
Next, we'll convert our custom `WSFederationAuthenticationModule` to use the `WsFederationAuthenticationMiddleware` from the OWIN pipeline.

```csharp
public class CustomWsFederationAuthenticationModule : WSFederationAuthenticationModule
{
  protected override void InitializeModule( HttpApplication context )
  {
      base.InitializeModule( context );

      RedirectingToIdentityProvider += OnRedirectingToIdentityProvider;
  }

  protected override void InitializePropertiesFromConfiguration()
  {
      Issuer = InstanceWideSettings.BaseStsUrl;
  }

  private void OnRedirectingToIdentityProvider( object sender, RedirectingToIdentityProviderEventArgs args )
  {
      // setting the realm in the OnRedirecting event allows it to be dynamic for multi-tenant applications
      args.SignInRequestMessage.Realm = Settings.BaseUrl;
  }
}
```
The code above will be removed and replaced with the `UseWSFederationAuthentication` helper below

```csharp
public void Configuration(IAppBuilder app)
{
   ...
   app.UseWsFederationAuthentication( new WsFederationAuthenticationOptions
   {
     // Pulls in STS Url and other metadata (like signing certificates)
     MetadataAddress = Settings.StsMetadataUrl,
     Notifications = new WsFederationAuthenticationNotifications
     {
         // replaces the OnRedirectingToIdentityProvider event
         RedirectToIdentityProvider = notification =>
         {
           notification.ProtocolMessage.Wtrealm = Settings.PresentationUrlRoot;
         }
     };
     // Name this authentication type (for WIF)
     AuthenticationType = WsFederationAuthenticationDefaults.AuthenticationType,
     // Tells the pipeline to use a cookie authenication we configured above to store the WIF session
     SignInAsAuthenticationType = CookieAuthenticationDefaults.AuthenticationType
   } );
}
```

### Move Global.asax.cs WSFederation configuration into OWIN configuration
Now that we've converted the two WSFederation HttpModules we can finish configuring the OWIN pipeline by converting either the WSFederation configuration in the web.config or that was configured on application start. In my case, I preferred to set up WSFederation in code using the `FederationConfigurationCreated` event like the code below:

```csharp
FederatedAuthentication.FederationConfigurationCreated += ( sender, args ) =>
{
  args.FederationConfiguration.IdentityConfiguration.AudienceRestriction.AudienceMode = ystem.IdentityModel.Selectors.AudienceUriMode.Always;

  // this method loads the list of relying parties for a multi-tenant application.
  List<string> relyingParties = GetRelyingParties();
  relyingParties.ForEach( rp => args.FederationConfiguration.IdentityConfiguration.AudienceRestriction.AllowedAudienceUris.Add( new Uri( rp  ) );

  // This code loads the metadata url, parses it and and updates the configuration with details from it like the signing certificates
  args.FederationConfiguration.IdentityConfiguration.IssuerNameRegistry = new CustomMetadataParser( Settings.StsMetadataUrl );
};
```

The items configured above can be added to the `UseWsFederationAuthentication` configuration:

```csharp
public void Configuration(IAppBuilder app)
{
   ...
   app.UseWsFederationAuthentication( new WsFederationAuthenticationOptions
   {
     ...
     TokenValidationParameters = new TokenValidationParameters()
     {
         // this replaces the IdentityConfiguration.AudienceRestriction setup
         ValidAudiences = GetRelyingParties(),
         ValidateAudience = true
     },
     // Pulls in STS Url and other metadata (like signing certificates) so we don't have to do custom metadata parsing
     MetadataAddress = Settings.StsMetadataUrl,
     ...
   } );
}
```

Additionally, in the Global.asax.cs file if you wanted to have access to WSFederation events you could declare special methods on your `HttpApplication` class and those would be invoked while the WSFederation protocol was executing. Two examples that I've used are shown below:

```csharp
void WSFederationAuthenticationModule_SessionSecurityTokenCreated( object sender, SessionSecurityTokenCreatedEventArgs e )
{
   // extend the expiration of the session cookie to make it last 1 year
   TimeSpan expiration = TimeSpan.FromYears( 1 );
   e.SessionToken = new SessionSecurityToken( e.SessionToken.ClaimsPrincipal, e.SessionToken.Context, now, now.Add( expiration ) ) { IsPersistent = true };

   e.WriteSessionCookie = true;
}

void WSFederationAuthenticationModule_RedirectingToIdentityProvider( object sender, RedirectingToIdentityProviderEventArgs e )
{
   // add client id parameter to outgoing wsfederation request
   e.SignInRequestMessage.Parameters.Add( "client_id", Settings.ClientId );
}
```

Again, these items can be replicated in the `UseWsFederationAuthentication` configuration:

```csharp
public void Configuration(IAppBuilder app)
{
   ...
   app.UseWsFederationAuthentication( new WsFederationAuthenticationOptions
   {
     ...
     Notifications = new WsFederationAuthenticationNotifications
     {
         // replaces the WSFederationAuthenticationModule_RedirectingToIdentityProvider method
         RedirectToIdentityProvider = notification =>
         {
           notification.ProtocolMessage.Parameters.Add( "client_id", Settings.ClientId );
         },
         // replaces the WSFederationAuthenticationModule_SessionSecurityTokenCreated method
         SecurityTokenValidated = notification =>
         {
           var newAuthenticationProperties = new AuthenticationProperties( authenticationTicket.Properties.Dictionary );

           DateTime now = DateTime.UtcNow;
           TimeSpan expiration = TimeSpan.FromYears( 1 );

           newAuthenticationProperties.IssuedUtc = now;
           newAuthenticationProperties.ExpiresUtc = now.Add( expiration );
           authenticationProperties.IsPersistent = true;

           return new AuthenticationTicket( claimsIdentity, authenticationProperties );
         }
     };
     ...
   } );
}
```

### Wrap up
At this point, all old WSFederation code is replaced and WSFederation actions are handled using the OWIN pipeline. One thing to note - we are not able to re-use existing sessions so existing user sessions will be invalidated by this code change. Once the user logs in again at the STS they'll be issued a new cookie that will work with the OWIN pipeline cookie authentication code.
