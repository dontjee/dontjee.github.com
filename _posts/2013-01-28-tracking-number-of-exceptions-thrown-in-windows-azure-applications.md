---
layout: post
title: "Tracking Number of Exceptions Thrown In Windows Azure Applications"
description: ""
category: 
tags: []
---
{% include JB/setup %}
Tracking performance counters is A Windows Azure best practice, and can provide insight into your application. The Windows Azure team [provides a list of all available performance counters](http://msdn.microsoft.com/en-us/library/windowsazure/hh411520.aspx). One counter that I've always found interesting is the number of exceptions thrown per second. A .NET application will have a certain baseline amount during normal operation, but if something is wrong this counter will often shoot up. However following their documentation, exceptions per second are not tracked.

After a lot of troubleshooting with Microsoft support, it turns out the fix to make it work is simple. Just change the counter from 

```
.NET CLR Exceptions(_Global_)/# Exceps Thrown / sec
```

to

```
.NET CLR Exceptions(_Global_)/# of Exceps Thrown / sec
```

Note the addition of the "of" in the second line. I hope this helps save someone else some time until Microsoft updates their documentation.
