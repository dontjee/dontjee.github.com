---
layout: post
title: "Calling Entity Framework Migrations From Code"
description: ""
category: 
tags: []
---
{% include JB/setup %}
Entity Framework Code First makes generating database migrations dead simple. They can then be run from the commandline with the `migrate.exe` executable included in the Entity Framework nuget package. However, in some continuous deployment scenarios it's easier to execute the migration from C# code. It turns out this process is pretty simple too, but not as well documented.

##### Running migrations from code
The three necessary classes are `DbConnectionInfo`, your specific implementation of `DbMigrationsConfiguration`, and `DbMigrator`. First, the `DbConnectionInfo` has to be configured
{% highlight csharp %}
string sqlConnectionString = "YOUR_CONNECTION_STRING_HERE";
var connectionInfo = new DbConnectionInfo( sqlConnectionString, "System.Data.SqlClient" );
{% endhighlight %}
The only tricky part is the second parameter. It's the "InvariantName" of the SQL provider that should be used with the provided connection string. `"System.Data.SqlClient"` is the provider for SQL Server according to the MSDN documentation.

Next, the `DbMigrationsConfiguration` must be set up
{% highlight csharp %}
var configuration = new MyDbMigrationConfiguration
                    {
                       TargetDatabase = connectionInfo,
                       AutomaticMigrationsEnabled = false,
                       AutomaticMigrationsDataLossAllowed = false
                    };
{% endhighlight %}
In the code above I disabled automatic migrations. For my continuous deployment scenario I want to be able to control the migrations explicitly, so I've disabled the `AutomaticMigration` features.

Finally, run the migrations using a `DbMigrator` instance
{% highlight csharp %}
var migrator = new DbMigrator( configuration );

// Roll back all migrations
migrator.Update( "0" );

// Roll back to a specific migration
migrator.Update( "MigrationNameToRollBackTo" );

// Apply all migrations up to a specific migration
migrator.Update( "MigrationNameToApply" );

// Apply all migrations
migrator.Update();
{% endhighlight %}
As you can see above the `Update` method gives you a lot of flexibility to control the migrations applied. One thing worth noting is that the string argument of the `Update` method is the class name of the migration, not the file name. This is important because they are usually different. The filename usually has a timestamp prepended to the name, while the class name does not.
