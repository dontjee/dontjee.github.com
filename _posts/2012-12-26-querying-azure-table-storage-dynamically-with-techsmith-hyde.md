---
layout: post
title: "Querying Azure Table Storage Dynamically With TechSmith Hyde"
description: ""
category: 
tags: []
---
{% include JB/setup %}
Lately I've been working on [TechSmith Hyde](http://techsmith.github.com/hyde/) a fair bit at work. One interesting workflow that Hyde allows you to accomplish is to query records via dynamics.
{% highlight csharp %}
    var tableStorage = new AzureTableStorageProvider(storageAccount);
    dynamic result = tableStorage.Get("MyTable", "Partition", "Row");
    Console.WriteLine(result.DynamicStringProperty);
{% endhighlight %}

All operations (Add/Insert/Update/Upsert/Delete) are supported with dynamics as well.
{% highlight csharp %}
    dynamic dog = new ExpandoObject();
    dog.Type = "Dog";
    dog.IsFurry = true;
    dog.Description = "Dogs have four legs";
    tableStorage.Add("AnimalsTable", dog.Type, Guid.NewGuid(), dog);

    dynamic specificDog = new ExpandoObject();
    specificDog.Type = "Dog";
    specificDog.Name = "Fido";
    specificDog.Age = 4;
    
    tableStorage.Add("AnimalsTable", specificDog.Type, specificDog.Name, specificDog);
    tableStorage.Save();
{% endhighlight %}

Using this pattern, multiple types of records can be stored in the same partition without needing to declare models for each record type. This is good for two reasons, first records stored in the same partition can be accessed with Entity Group Transactions. This is useful if you want to modify several different records at the same time. The second reason is flexibility. For example, with a tool like LINQPad you can query Table Storage without knowing what properties a row might contain.

Finally, this opens up the possibility to query those different record types at runtime and handle them separately based on what properties are definied. 
{% highlight csharp %}
    dynamic result = tableStorage.GetCollection("AnimalsTable", "Dog");
    foreach(dynamic record in result)
    {
       try
       {
          string description = record.Description;
          HandleAnimalType(record);
       }
       catch(RuntimeBinderException)
       {
          HandleAnimalInstance(record);
       }
    }
{% endhighlight %}

This code will query all records we have for the `"Dog"` partition and process them, handling both the animal type and animal instance records. This could be even be expanded to handle multiple different types of records with the same query. The benefit is that it only takes 1 query to bring back all records. 

Dynamic querying of Table Storage is not for every situation, but it can be a powerful tool in your toolbox.
