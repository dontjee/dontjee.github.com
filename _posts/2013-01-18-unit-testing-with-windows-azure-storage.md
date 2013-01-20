---
layout: post
title: "Unit Testing with Windows Azure Storage"
description: ""
category: 
tags: []
---
{% include JB/setup %}
I'm of the opinion that [good developers write unit tests](http://www.codinghorror.com/blog/2006/07/i-pity-the-fool-who-doesnt-write-unit-tests.html). This means when developing against Windows Azure Table Storage, responsible developers will eventually stumble onto the question of how to unit test code written using the Azure SDK. The common answer is to wrap access to table storage in a [Repository Object](http://www.remondo.net/repository-pattern-example-csharp/) or [Query Object](http://lostechies.com/jimmybogard/2012/10/08/favor-query-objects-over-repositories/). These objects help by abstracting the rest of the code from the nitty gritty details of talking to table storage. Then when unit testing, [replace those objects with a test double](http://www.martinfowler.com/bliki/TestDouble.html) to keep your code from actually trying to query table storage.

That's great for the rest of the code, but what about testing the Repository/Query Objects? Integration tests are used to test the objects against real table storage. Others may skip this step thinking there shouldn't be much logic to worry about in these objects.

This approach works but I think we can do better. No matter how hard we try, there is always going to be some amount of logic in our data access classes. This is where TechSmith Hyde comes in.

#####Using TechSmith Hyde to unit test access to Azure Table Storage
The approach that TechSmith Hyde takes is to provide a [fake implementation of table storage](http://www.martinfowler.com/bliki/InMemoryTestDatabase.html) that implements the {% highlight csharp %}TableStorageProvider{% endhighlight %} base class that the Azure implementation does. This allows the developer to write methods against {% highlight csharp %}TableStorageProvider{% endhighlight %} and select the test/production implementation with dependency injection with consistent behavior regradless of the implementation. A simple example is below:

{% highlight csharp %}
public class UserRepository
{
  TableStorageProvider _table;
  public UserRepository(TableStorageProvider table)
  {
    _table = table;
  }

  public User GetUser(string id)
  {
    return _table.Get<User>("UsersTable", id, id);
  }

  // Other methods ommitted...
}

[TestClass]
public class UserRepositoryTest
{
  [TestMethod]
  public void GetUser_UserExistsInTable_ReturnsCorrectUser()
  {
    var expectedUser = new User{ Id = "123", Name = "Tyler Durden" };

    var table = new InMemoryTableStorageProvider();
    table.Add("UsersTable", expectedUser.Id, expectedUser.Id);

    var userRepository = new UserRepository(table);

    var actualUser = userRepository.GetUser("123");

    Assert.AreEqual(expectedUser.Name, actualUser.Name);
  }
}
{% endhighlight %}

As you can see in the simple test above we're able to test table storage access using an in-memory fake and in production inject the Table Storage implementation instead. Using Hyde we can test our data access classes, increasing test coverage in our application and allowing us to feel more confident our code does what we think it does.
