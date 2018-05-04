---
layout: "post"
title: "Full Framework WSFederation to OWIN Conversion"
description: ''
category: null
tags: []
---










In my last post I suggested using the [database stream writer pattern](/2017/12/13/avoid-dual-writes-with-sql-server-and-change-tracking-part-1/) to avoid writing to multiple data stores outside of a transaction from your application. This post will detail the implementation. For my example the application is a  c# application writing to SQL Server and tracking changes using the Change Tracking feature. The data model for this example is similar to Youtube. It contains users, channels and media. A user has one or more channels which in turn have one or more pieces of media associated with them. The full schema is contained in [this gist](https://gist.github.com/dontjee/aebaabd7737f113382a6f0384015232c).

At a high level, there are 3 pieces of the architecture to consider.
1. The primary application which will write to the SQL database. It will not deal with writes to the downstream data stores and will largely be ignored by this post.
1. The SQL Server database which will be configured to track changes to all necessary tables.
1. The application to monitor the change tracking stream from the database and push updates to their datastore (cache, search, etc).

#### Database Implementation

First [change tracking](https://docs.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-tracking-sql-server) must be enabled on the SQL Server database and tables. The SQL below enables change tracking on all three tables configured to retain changes for seven days and enables automatic cleanup of expired changes.
```sql
ALTER DATABASE BlogPostExample
SET CHANGE_TRACKING = ON  
  (CHANGE_RETENTION = 7 DAYS, AUTO_CLEANUP = ON)  

ALTER TABLE dbo.UserAccount
ENABLE CHANGE_TRACKING  
WITH (TRACK_COLUMNS_UPDATED = ON)

ALTER TABLE dbo.Channel
ENABLE CHANGE_TRACKING  
WITH (TRACK_COLUMNS_UPDATED = ON)

ALTER TABLE dbo.Media
ENABLE CHANGE_TRACKING  
WITH (TRACK_COLUMNS_UPDATED = ON)
```

One final table is required to track the current position in the change tracking stream our monitor has consumed. The following SQL will create the table and initialize it with the minimum change tracking version currently in the database:

```sql
CREATE TABLE [dbo].[CacheChangeTrackingHistory] (
   [CacheChangeTrackingHistoryId]               INT   IDENTITY (1, 1) NOT NULL,
   [TableName]                             NVARCHAR (512)   NOT NULL,
   [LastSynchronizationVersion]            BIGINT   NOT NULL,
);
ALTER TABLE [dbo].[CacheChangeTrackingHistory]
   ADD CONSTRAINT [PK_CacheChangeTrackingHistory] PRIMARY KEY CLUSTERED ([CacheChangeTrackingHistoryId] ASC);

-- Add default values for last sync version
INSERT INTO dbo.CacheChangeTrackingHistory( TableName, LastSynchronizationVersion )
VALUES ('dbo.UserAccount', CHANGE_TRACKING_MIN_VALID_VERSION(Object_ID('dbo.UserAccount')))
INSERT INTO dbo.CacheChangeTrackingHistory( TableName, LastSynchronizationVersion )
VALUES ('dbo.Channel', CHANGE_TRACKING_MIN_VALID_VERSION(Object_ID('dbo.Channel')))
INSERT INTO dbo.CacheChangeTrackingHistory( TableName, LastSynchronizationVersion )
VALUES ('dbo.Media', CHANGE_TRACKING_MIN_VALID_VERSION(Object_ID('dbo.Media')))
```

#### Change Monitor Implementation

With the database properly configured we can start on the application that will consume the change tracking stream from SQL Server. The changes can be accessed by the `CHANGETABLE( CHANGES <TABLE_NAME> )` function. I will focus on `UserAccount` changes but the code will apply equally to the `Channel` and `Media` tables. When our monitor application starts, a loop is started to process change tracking updates and push them to downstream data stores. In this case the only downstream data store is the Cache represented by the ICache interface. If we had multiple downstream systems to update, the application would start one monitoring loop with a distinct change tracking history table for each system.

```java
public static class ChangeTracker
{
    internal static async Task StartChangeTrackingMonitorLoopAsync( CancellationToken token, ICache userAccountCache )
    {
       while ( true )
       {
          if ( token.IsCancellationRequested )
          {
             break;
          }

          using ( ChangeTrackingBatch<UserAccountChangeModel> userAccountChangesBatch = GetLatestUserChanges() )
          {
             UserAccountChangeModel[] userAccountChanges = ( await userAccountChangesBatch.GetItemsAsync() ).ToArray();
              foreach( var userAccount in userAccountChanges )
              {
                userAccountCache.UpdateObject( "user_account", userAccount.OperationType, userAccount.UserAccountId, userAccount );
              }
              userAccountChangesBatch.Commit();
          }

          await Task.Delay( 1000 );
       }
    }

    private Task<IEnumerable<UserAccountChangeModel>> GetLatestUserChangesAsync()
    {
         string cmd = "
DECLARE @last_synchronization_version BIGINT = (SELECT LastSynchronizationVersion FROM dbo.CacheChangeTrackingHistory WHERE TableName = 'dbo.UserAccount')

DECLARE @current_synchronization_version BIGINT = CHANGE_TRACKING_CURRENT_VERSION();
SELECT ct.UserAccountId, ua.Email, ua.DisplayName, ua.CreateDate
		, CASE WHEN ct.SYS_CHANGE_OPERATION = 'I' THEN 'Insert' WHEN ct.SYS_CHANGE_OPERATION = 'U' THEN 'Update' ELSE 'Delete' END AS OperationType
FROM dbo.UserAccount AS ua
	RIGHT OUTER JOIN CHANGETABLE(CHANGES dbo.UserAccount, @last_synchronization_version) AS ct ON ua.UserAccountId = ct.UserAccountId

UPDATE dbo.CacheChangeTrackingHistory
SET LastSynchronizationVersion = @current_synchronization_version
WHERE TableName = 'dbo.UserAccount'
";
         return new ChangeTrackingBatch<UserAccountChangeModel>( _connectionString, cmd );
    }
}
```

```csharp
internal class ChangeTrackingBatch<T> : IDisposable
{
  private readonly string _command;
  private SqlTransaction _transaction;
  private IEnumerable<T> _items;
  private SqlConnection _connection;
  private readonly object _param;

  public ChangeTrackingBatch( string connectionString, string command, object param = null )
  {
     _connection = new SqlConnection( connectionString );
     _command = command;
     _param = param;
  }

  public async Task<IEnumerable<T>> GetItemsAsync( )
  {
     if ( _items != null )
     {
        return _items;
     }

     _connection.Open();
     _transaction = _connection.BeginTransaction( System.Data.IsolationLevel.Snapshot );
     _items = await _connection.QueryAsync<T>( _command, _param, _transaction );
     return _items;
  }

  public void Commit()
  {
     _transaction?.Commit();
     _connection?.Close();
  }

  public void Dispose()
  {
     Dispose( true );
     GC.SuppressFinalize( this );
  }

  protected virtual void Dispose( bool disposing )
  {
     if ( disposing )
     {
        _transaction?.Dispose();

        _connection?.Dispose();
     }
  }
}
```

One additional thing to note is the use of [SnapshotIsolationMode](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/sql/snapshot-isolation-in-sql-server) for the database transaction. This makes sure we're working with a consistent view of the database to prevent any collisions with the change tracking cleanup process.

#### Wrap Up
At this point the solution is complete. Any updates to the `UserAccount` table will be tracked and pushed into the cache by the change tracker class. If the update to the cache fails, the transaction will be rolled back. The monitor application will then retry applying changes in order until the change is pushed into the cache successfully or the change is cleaned up by change tracking retention settings.

This solution is tied to the scalability of SQL Server, so for a write heavy application a different architecture may be necessary. For example, SQL Server could be replaced by a log stream like [Kafka](https://kafka.apache.org). However, for moderate scale applications this architecture will be more than adequate to handle load. We've solved the resiliency and race condition issues from the dual write scenario by ensuring that any successful database write will be pushed to downstream systems, in order. We've also improved the overall architecture of the primary application by removing the writes to secondary data stores. Plus we haven't introduced any new data stores to learn and manage. For reference, [the full application source code is available on GitHub](https://github.com/dontjee/WriteThroughDatabaseToCache).
