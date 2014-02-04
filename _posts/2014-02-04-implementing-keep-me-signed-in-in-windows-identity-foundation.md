---
layout: post
title: "Implementing 'Keep Me Signed In' in Windows Identity Foundation"
description: ""
category: 
tags: []
---
{% include JB/setup %}
A common feature of website authentication is the 'Remember me' or 'Keep me signed in' option. This feature is not a built-in feature of Windows Identity Foundation. The easiest solution is to make all Relying Party cookies Session cookies, meaning they expire when you close the browser. When you navigate back to the relying party you'll be sent to the STS, automatically logged in and sent back. This can be a pain for a number of reasons so it's ideal if we can setup the Relying Party cookies the same as the STS. I'll show how it can be implemented using claims as the means of communication between the STS and Relying Party.

####The STS setup
To communicate whether or not the user wanted to be remembered, we're going to use claims. Specifically we'll be using two existing claims from the `Microsoft.IdentiyModel.Claims` namespace,  `IsPersistent` and `Expiration`. To do so, first add the claims to the FederationMetadata xml so you see something like this:

{% highlight xml %}
<auth:ClaimType xmlns:auth="http://docs.oasis-open.org/wsfed/authorization/200706" Uri="http://schemas.microsoft.com/ws/2008/06/identity/claims/ispersistent" Optional="true">
   <auth:DisplayName>isPersistent</auth:DisplayName>
   <auth:Description>If subject wants to be remembered for login.</auth:Description>
</auth:ClaimType>
<auth:ClaimType xmlns:auth="http://docs.oasis-open.org/wsfed/authorization/200706" Uri="http://schemas.microsoft.com/ws/2008/06/identity/claims/expiration" Optional="true">
   <auth:DisplayName>Expiration</auth:DisplayName>
   <auth:Description>How long before the persistent session cookie should expire</auth:Description>
</auth:ClaimType>
{% endhighlight %}

As the description states, we'll be using the `IsPersistent` claim to communicate if the user wanted to be kept logged in and the `Expiration` claim to communicate the session expiration if `IsPersistent` is true.

The last step on the Relying Party is to set the claims on the user's principal. Update the `IClaimsPrincipal` creation code to specify the two new claims.

{% highlight csharp %}
public static IClaimsPrincipal CreatePrincipal( UserModel user, bool rememberMe )
{
  if ( user == null )
  {
    throw new ArgumentNullException( "user" );
  }

  // CLAIMS ADDED HERE SHOULD MATCH WITH CLAIMS OFFERED BY METADATA
  var claims = new List<Claim>
  {
    ... // Your other claims go here
    new Claim( ClaimTypes.IsPersistent, rememberMe.ToString() ),
    new Claim( ClaimTypes.Expiration, TimeSpan.FromDays( DEFAULT_COOKIE_EXPIRATION_IN_DAYS ).ToString() )
  };

  var identity = new ClaimsIdentity( claims );

  return ClaimsPrincipal.CreateFromIdentity( identity );
}
{% endhighlight %}

The two steps above ensure that the STS will communicate the necessary information to the Relying Party for them to set up their session to mirror the STS session.

####Relying Party setup

On the Relying Party side we have to override the default WIF behavior for the session expiration and set it manually based on the claims we've specified in the STS. We'll need to override the `SessionSecurityTokenCreated` behavior to do so. Place the following code in the `global.asax` of the Relying Party.

{% highlight csharp %}
// This method does not appear to be used, but it is.
// WIF detects it is defined here and calls it.
// Note: Do not rename this method. The name must exactly match or it will not work.
[System.Diagnostics.CodeAnalysis.SuppressMessage( "Microsoft.Performance", "CA1811:AvoidUncalledPrivateCode" )]
void WSFederationAuthenticationModule_SessionSecurityTokenCreated( object sender, SessionSecurityTokenCreatedEventArgs e )
{
  bool isPersistent = false;
  string expirationAsString = null;
  try
  {
    isPersistent = ClaimsHelper.GetClaimValueByTypeFromPrincipal<bool>( e.SessionToken.ClaimsPrincipal, ClaimTypes.IsPersistent );
    expirationAsString = ClaimsHelper.GetClaimValueByTypeFromPrincipal<string>( e.SessionToken.ClaimsPrincipal, ClaimTypes.Expiration );
  }
  catch ( ClaimParsingException )
  {
    Trace.TraceWarning( "Failure to parse claim values for ClaimTypes.IsPersistent and ClaimTypes.Expiration. Using session cookie as a fallback." );
  }
  catch ( ClaimNullException )
  {
    Trace.TraceWarning( "Expected claim values for ClaimTypes.IsPersistent and ClaimTypes.Expiration but got null. Using session cookie as a fallback." );
  }

  TimeSpan expiration;
  if ( isPersistent && TimeSpan.TryParse( expirationAsString, CultureInfo.InvariantCulture, out expiration ) )
  {
    DateTime now = DateTime.UtcNow;
    e.SessionToken = new SessionSecurityToken( e.SessionToken.ClaimsPrincipal, e.SessionToken.Context, now, now.Add( expiration ) )
    {
      IsPersistent = true
    };
  }
  else
  {
    e.SessionToken = new SessionSecurityToken( e.SessionToken.ClaimsPrincipal, e.SessionToken.Context )
    {
      IsPersistent = false
    };
  }
  e.WriteSessionCookie = true;
}
{% endhighlight %}

The important part is at the end. We create a new `SessionSecurityToken` object based on the values of the claims and overwrite the default WIF security token with it. This gives us either a session cookie or a cookie with an expiration that matches the STS value; giving us the 'Keep me logged in' behavior we wanted.
