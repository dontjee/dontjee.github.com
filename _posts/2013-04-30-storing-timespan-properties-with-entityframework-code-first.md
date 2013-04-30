---
layout: post
title: "Storing TimeSpan Properties with EntityFramework Code First"
description: ""
category: 
tags: []
---
{% include JB/setup %}
Entity Framework 5 Code First does a good job of selecting a corresponding SQL column type for most C# primitives. However, the type it chooses for `TimeSpan` properties can cause problems. It chooses the [Time type](http://msdn.microsoft.com/en-us/library/bb677243.aspx) which can only store ranges up to 24 hours. If your `TimeSpan` needs to store more than 24 hours, you need to choose a different option.

The strategy I've found most useful is to store the data as Ticks in a `BIGINT` column. You can achieve this by using the code below

{% highlight csharp %}
[NotMapped]
public TimeSpan TimeToCompleteForm
{
  get;
  set;
}

public long TimeToCompleteFormTicks
{
  get
  {
    return TimeToCompleteForm.Ticks;
  }
  set
  {
    TimeToCompleteForm = TimeSpan.FromTicks( value );
  }
}
{% endhighlight %}

In SQL you can query this value as raw ticks or convert it to a readable string in the format `'dd.hh:mm:ss:ms'` using the following query:
{% highlight sql %}
SELECT CONVERT(VARCHAR, DATEPART(DAY,DATEADD(ms, TimeToCompleteFormTicks/10000, 0))) + '.' + CONVERT(VARCHAR, DATEADD(ms, TimeToCompleteFormTicks/10000, 0), 114)
{% endhighlight %}

Finally, if you prefer to represent the value in milliseconds instead of ticks, the code above requires two tweaks. Instead of `Ticks` use the `TotalMilliseconds` property of the `TimeSpan`. Additionally, use the `FromMilliseconds` method to convert the incoming milliseconds to a `TimeSpan` instead of `FromTicks`.
