---
layout: post
title: "Finding VM IP Address in a Windows Azure startup task"
description: ""
category: 
tags: []
---
{% include JB/setup %}

In a Windows Azure startup task, there are a few options for finding a server's IP address.

One way is to add an environment variable via the csdef as follows:
{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<ServiceDefinition name="AzureIPAddressExample" xmlns="http://schemas.microsoft.com/ServiceHosting/2008/10/ServiceDefinition">
  <WorkerRole name="IPAddressExample" vmsize="Small">
    <Runtime>
      <Environment>
        <Variable name="ADDRESS">
          <RoleInstanceValue xpath="/RoleEnvironment/CurrentInstance/Endpoints/Endpoint[@name='HttpIn']/@address" />
        </Variable>
      </Environment>
    </Runtime>
    <Endpoints>
      <InputEndpoint name="HttpIn" protocol="tcp" port="8080" />
    </Endpoints>
  </WorkerRole>
</ServiceDefinition>
{% endhighlight %}

Then in a startup task you can access the variable like a normal enviroment variable in a powershell or batch script:
{% highlight bat %}
echo IP Address %ADDRESS%
{% endhighlight %}

That works great on any type of Azure role but requires a change to the csdef. If you know you only need to get the IP for a web role, you can grab the IP from the IIS bindings with powershell.

{% highlight ps1 %}
$appcmd="$env:windir\system32\intesrv\appcmd"

$bindingText = & $appcmd list sites /text:bindings /name:"$=*YourSiteName"

$bindingText -match "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"

$ipAddress = $matches[0]
{% endhighlight %}

How this works is by grabbing the bindings for your site from IIS via the appcmd exe. Next feed the bindings string into a regex and extract the IP Address.

Both options work, but my preference is to use the second when I'm in a web role because it doesn't require any changes to the .csdef.
