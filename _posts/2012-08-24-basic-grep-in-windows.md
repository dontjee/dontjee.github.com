---
layout: post
title: "Basic Grep in Windows"
description: ""
category: 
tags: []
---
{% include JB/setup %}
When developing on Windows machines I often find myself needing to search through a group of files looking for a given string. The standard tool for the job is `grep`, and while it can be installed on Windows it isn't installed by default. Instead I've found that the `findstr` command performs 90% of what I need.

<br />

For example the following will look through the current directory and all subdirectories for the string `thetexttofind`
{% highlight bat %}
findstr -s "thetexttofind" *.*
{% endhighlight %}

You can also search for regular expressions. The following example will search for all lines in a file that start with the string `foo`
{% highlight bat %}
findstr -s "^foo.*" *.*
{% endhighlight %}

<br />

If you prefer Powershell for Windows scripting, these commands work there as well.
