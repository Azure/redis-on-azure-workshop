# Streams in Azure Cache for Redis

## Learning Objective
In this exercise, you will implement Streams in an Azure Cache for Redis instance.

* Use the Redis console to create, append, and query a Redis stream with time-based ranges.
* Append and query entries in a Redis stream from a .NET Core application.

> If your application requires stronger delivery guarantees, you may want to learn about Redis Streams. Messages in streams are persisted, and support both *at-most-once* as well as *at-least-once* delivery semantics. Streams also support consumer groups, which allow multiple consumers to read from the same stream. For more information, see [Redis Streams](https://redis.io/topics/streams-intro).

## Prerequisites

* Azure Cache for Redis instance - If you deleted your instance from the previous lab, please follow the steps [here](./labs/01-explore-azure-cache-for-redis.md#create-an-azure-cache-for-redis-using-the-azure-cli) to create a new one.

## Add entries to a stream
Entries are added to a new or existing stream using the ```XADD``` command. The stream is automatically created if it does not already exists.

1. Sign in to the [Azure portal](https://portal.azure.com/#view/HubsExtension/BrowseResource/resourceType/Microsoft.Cache%2FRedis). Navigate to your Azure Cache for Redis instance.

1. In the Overview pane, select **Console**. This will open a Redis Console, which enables you to enter low-level Redis commands.

    ![Redis Console Menu](./assets/redis-console-menu.png)

1. In the console, use the ```XADD``` command to add two entries to the ```org.logs.clientapp``` stream:

    ```redis
    XADD org.logs.clientapp 1324092248593-0 device-id mobile error unknown-crash
    XADD org.logs.clientapp 1481945061467-0 worker-process 1788 status success
    ```

    > The first argument to the ```XADD``` command is the name of the stream. The second argument is the key of the entry. The key is a combination of the current Unix time in milliseconds and a sequence number. The remaining arguments are the fields and values of the entry.

1. Observe the output from the two invocations of the ```XADD``` command. The output will include the key of the newly added entries.

    ```redis
    "1324092248593-0"
    "1481945061467-0"

1. Use the ```XADD``` command to add another new entry with an automatically generated identifier:

    ```redis
    XADD org.logs.clientapp * application-status started
    ```

    > The ```*``` argument to the ```XADD``` command indicates that the key should be automatically generated.

1. Observe the output from the invocation of the ```XADD``` command. The output includes a newly generated key based on the current Unix time in milliseconds and a sequence number. For example, if the key is "1638502526759-0", then the output would be:

    ```redis
    "1638502526759-0"
    ```

## Retrieve and count all entires in a stream
The ```XLEN``` command counts the number of entries in a stream. Once you are ready to query the entries, you can use the ```XRANGE``` command to get entries within the stream.

1. Use the ```XLEN``` command to count the number of entries in the ```org.logs.clientapp``` stream:

    ```redis
    XLEN org.logs.clientapp
    ```

1. Observe the output from the ```XLEN``` command. The output will be an integer with a value of 3 for the entries created earlier in this exercise.

    ```redis
    (integer) 3
    ```

1. Use the ```XRANGE``` command and both the ```+``` and ```-``` operators to get a range of all data in the ```org.logs.clientapp``` stream:

    ```redis
    XRANGE org.logs.clientapp - +
    ```

1. Observe the output from invoking the ```XRANGE``` command. This output will include all three entries in the stream.

    ```redis
    1) 1) "1324092248593-0"
       2) 1) "device-id"
          2) "mobile"
          3) "error"
          4) "unknown-crash"
    2) 1) "1481945061467-0"
       2) 1) "worker-process"
          2) "1788"
          3) "status"
          4) "success"
    3) 1) "1638502526759-0"
       2) 1) "application-status"
          2) "started"
    ```

    > Your last key will not exactly match the identifier used in this example.

1. The ```XRANGE``` command includes a ```+``` and ```-``` operator. These operators can be used with keys to query a subset of the data in a stream based on a time range.

1. Invoke the XRANGE command using the - operator and the key of the second entry (1481945061467-0):

    ```redis
    XRANGE org.logs.clientapp - 1481945061467-0
    ```

1. Observe the output of the invocation of the ```XRANGE``` command. The output includes all entries from the start of the stream, chronologically, up to the second entry (1481945061467-0).

    ```redis
    1) 1) "1324092248593-0"
       2) 1) "device-id"
          2) "mobile"
          3) "error"
          4) "unknown-crash"
    2) 1) "1481945061467-0"
       2) 1) "worker-process"
          2) "1788"
          3) "status"
          4) "success"
    ```

1. Invoke the ```XRANGE``` command using the key of the second entry (1481945061467-0) and the + operator:

    ```redis
    XRANGE org.logs.clientapp 1481945061467-0 +
    ```

1. Observe the output of the invocation of the ```XRANGE``` command. The output includes the second entry, and then all entries up to the end of the stream, chronologically.

    ```redis
    1) 1) "1481945061467-0"
       2) 1) "worker-process"
          2) "1788"
          3) "status"
          4) "success"
    2) 1) "1638502526759-0"
       2) 1) "application-status"
          2) "started"
    ```redis
    
    > The last key will not exactly match the one used in this example.

## Apend to and query streams from .NET Core

1. Create a new .NET Core console application and open the project in Visual Studio Code.

    ```bash
    dotnet new console --name redis-streams
    cd redis-streams
    code .
    ```

1. Add the NuGet package ```StackExchange.Redis``` using the terminal shell.:

    ```bash
    dotnet add package StackExchange.Redis
    ```

1. Update ```Program.cs``` to create a ```ConnectionMultiplexer```:

    ```csharp
    using StackExchange.Redis;
    
    var connectionString = "[cache-name].redis.cache.windows.net:6380,password=[password-here],ssl=True,abortConnect=False";
    var redisConnection = ConnectionMultiplexer.Connect(connectionString);
    ```

    > You can obtain the ```connectionString``` from **Access keys** section of the Azure Cache for Redis instance in the Azure portal.

1. The Redis database is represented by the ```IDatabase``` type. You can retrieve one using the ```GetDatabase()``` method:

    ```csharp
    IDatabase db = redisConnection.GetDatabase();
    ```

1. Use the following to add a simple message with a single name/value pair to a stream:

    ```csharp
    var messageId = db.StreamAdd("events_stream", "foo_name", "bar_value");
    Console.WriteLine($"messageId = {messageId}");
    // messageId = 1518951480106-0
    ```

    > Each message or entry in the stream is represented by the ```StreamEntry``` type. Each stream entry contains a unique ID and an array of name/value pairs. The name/value pairs are represented by the ```NameValueEntry``` type.

1. Multiple name/value pairs can be written to a stream using the following:

    ```csharp
    var values = new NameValueEntry[]
    {
        new NameValueEntry("sensor_id", "1234"),
        new NameValueEntry("temp", "19.8")
    };

    var sensorMessageId = db.StreamAdd("sensor_stream", values);
    Console.WriteLine($"sensorMessageId = {sensorMessageId}");
    // sensorMessageId = 1681829523719-0
    ```

1. Reading from a stream is done by using either the ```StreamRead``` or ```StreamRange``` methods. Read all messages from the ID "0-0" to the end of the stream.

    ```csharp
    var messages = db.StreamRead("events_stream", "0-0");
    var writeMessage = (string stream, StreamEntry message) => {
        Console.WriteLine($"stream = {stream}");
        Console.WriteLine($"messageId = {message.Id}");
        foreach (var entry in message.Values)
        {
            Console.WriteLine($"entry = {entry.Name}:{entry.Value}");
        }
    };

    foreach (var message in messages)
    {
        writeMessage("events_stream", message);
    }
    ```

1. The ```StreamRead``` method also allows you to read from multiple streams at once:

    ```csharp
    var streams = db.StreamRead(new StreamPosition[]
    {
        new StreamPosition("events_stream", "0-0"),
        new StreamPosition("sensor_stream", "0-0")
    });

    Console.WriteLine($"Stream = {streams.First().Key}");
    Console.WriteLine($"Length = {streams.First().Entries.Length}");
    foreach (var stream in streams)
    {
        foreach (var message in stream.Entries)
        {
            writeMessage(stream.Key.ToString(), message);
        }
    }
    ```

1. The ```StreamRange``` method allows you to return a range of entries within a stream.

    ```csharp
    db.StreamRange("events_stream", minId: "-", maxId: "+");
    ```

    > The "-" and "+" special characters indicate the smallest and greatest IDs possible. These values are the default values that will be used if no value is passed for the respective parameter.

## Additional Resources

1. [Implement Pub/Sub and Streams in Azure Cache for Redis](https://learn.microsoft.com/en-us/training/modules/azure-redis-publish-subscribe-streams/)