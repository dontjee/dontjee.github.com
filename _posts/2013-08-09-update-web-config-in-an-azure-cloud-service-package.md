---
layout: post
title: "Update Web.Config in an Azure Cloud Service package"
description: ""
category: 
tags: []
---
{% include JB/setup %}
Windows Azure deployments are done using a convenient `.cspkg` and `.cscfg` files. The `.cscfg` is an xml config file and the `.cspkg` is essentially a zip file that contains your application code. This means you can build once and deploy to different environments by providing a different version of the `.cscfg`, making continuous deployment simple. Just keep the `.cspkg` file around and deploy it anywhere.

Problems arise when you need to modify something in the cspkg, such as the `Web.Config` for your web application. A common scenario where this is necessary is configuring [Windows Identity Foundation](http://msdn.microsoft.com/en-us/security/aa570351.aspx) to update a trusted issuer thumbprint or federation realm. Options to fix the problem are to create a new package for each environment or creating the package on demand as part of the deploy process. Microsoft has [provided a way to create packages manually](http://msdn.microsoft.com/en-us/library/windowsazure/gg432988.aspx) but it's complicated to set up and involves duplicating a lot of work that's done for us already in the MSBuild tasks for the cloud project.

#####The fix
An alternate approach I've had success with is to modify the `Web.Config` on role start in your web project based on values stored in the `.cscfg` configuration file. To do this, copy the `Web.Config` in your project and rename it to `Web.Config_pretransform` or something similar. Also stop tracking the `Web.Config` in your source control since it will just be generated as needed (but make sure the project still has a reference to it). Next add code to your `WebRole.cs` to do the file modification like so:
{% highlight csharp %}
public override bool OnStart()
{
   UpdateConfigs();
}
{% endhighlight %}

Fill in the `UpdateConfigs` method with code to open the `Web.Config_pretransform` using `Microsoft.Web.Administration.ServerManager`.
{% highlight csharp %}
private void UpdateConfigs()
{
  using ( var server = new ServerManager() )
  {
    Site site = server.Sites[RoleEnvironment.CurrentRoleInstance.Id + "_Web"];
    string physicalPath = site.Applications["/"].VirtualDirectories["/"].PhysicalPath;
    string inputWebConfigPath = Path.Combine( physicalPath, "web.config_pretransform" );
    string outputWebConfigPath = Path.Combine( physicalPath, "web.config" );

    File.Copy( inputWebConfigPath, outputWebConfigPath, overwrite:true );

    SetWIFWebConfigSettings( outputWebConfigPath );
  }
}
{% endhighlight %}

The code above grabs the physical path of the website from IIS and passes it off to the `SetWIFWebConfigSettings` method. This method can then parse and update the Web.Config using your favorite XML parser. Finally, the code below shows how to update the `realm` and `requireHttps` attributes using values from the `.cscfg`:
{% highlight csharp %}
private static void SetWIFWebConfigSettings( string webConfigPath )
{
  var doc = XDocument.Load( webConfigPath );
  var wifConfig = doc.Descendants( "microsoft.identityModel" ).Single();

  var wsFederation = wifConfig.Descendants( "federatedAuthentication" ).Single()
    .Descendants( "wsFederation" ).Single();
  wsFederation.SetAttributeValue( "realm", CloudConfigurationManager.GetSetting( "WifWsFederationRealm" ) );
  wsFederation.SetAttributeValue( "requireHttps", "true" );

  doc.Save( webConfigPath );
}
{% endhighlight %}

* It's important to note that Azure will not actually put our instance on the load balancer until after the `RoleStart` method has finished which allows us to do these modifications.

One last thing we need to do is make it work locally as well. An easy fix is to copy the `Web.Config.pretransform` to the `Web.Config` location prior to building the project if it doesn't exist.
{% highlight xml %}
<PropertyGroup>
   <PreBuildEvent>if not exist "$(ProjectDir)\Web.config" (copy /Y "$(ProjectDir)\Web.config_pretransform" "$(ProjectDir)\Web.config") else (echo web.config already exists in $(ProjectDir), skipping)</PreBuildEvent>
</PropertyGroup>
{% endhighlight %}

That's all we need to do to modify the Web.Config on role start in an Azure Cloud Service. It lets us keep all environment specific settings in the `.cscfg` which means we can deploy one package to any environment.
