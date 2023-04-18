# RedisJSON in Azure Cache for Redis

JSON (JavaScript Object Notation) is an open standard file and data interchange format. If you've worked with any modern web application or API, you've probably used JSON.

JSON is supported by most programming languages, and typically native objects and arrays are encoded as JSON (serialized), before being sent over the network or written to a file. Parsing JSON, or deserialization, is the opposite of this process.

Storing JSON in Azure Cache for Redis Enterprise let's you work with JSON values directly, without the need for multiple operations to retrieve, update or access values within the data. When working with JSON in your cache, there's no need to serialize and deserialize data.

The RedisJSON module provides JSON support for Redis. RedisJSON lets you store, update, and retrieve JSON values in a Redis database, similar to any other Redis data type.

Azure Cache for Redis Enterprise includes RedisJSON. Once enabled, your cache becomes a powerful, superfast in-memory JSON document store.

## Learning Objective
Completing this lab will provide you with an understanding of:
- How to use Redis CLI commands for RedisJSON
- How to create and store JSON in Redis
- How to modify JSON using JSONPath
- How RedisJSON can be used in a .NET application

## Prerequisites
- Redis CLI or RedisInsight v2 (Microsoft Store [link](https://apps.microsoft.com/store/detail/redisinsight/XP8K1GHCB0F1R2))
- Redis Stack or Azure Cache for Redis Enterprise with the JSON module enabled
- .NET 6 (for the code examples)

## Steps

Lets imagine a scenario where we are working for an online food order company. Redis and Microsoft are running a developer meetup and have decided to order in some food 🍕 and refreshments 🥤 for the guests.

Let's consider what a typical JSON object might look like that contains the data for the order:

```json
{
   "orderNumber": 0,
   "deliveryAddress": "",
   "orderTotal": 0.0,
   "paid": false,
   "shipped": false,
   "items":[
      {
         "sku": "",
         "description": "",
         "quantity": 0,
         "price": 0
      }
   ]
}
```

To create JSON in Redis, use the `JSON.SET` command. For example, to create an empty object:

```
JSON.SET order $ '{}'
```

To check the value, use ```JSON.GET```:

```
JSON.GET order
```

Now let's delete it:

```
JSON.DEL order
```

We could create our example JSON document above using one command, but let's build up our object incrementally. Here we start with an empty object:

Using `JSON.SET`, you can choose to overwrite data or work with values using JSONPath (hence the `$` - this denotes the root of the JSON object). For example, let's start with this inital object containing a few properties:

```
JSON.SET order $ '{"deliveryAddress": "Microsoft UK, Thames Valley Park, Reading", "orderTotal": 0.00, "paid": false}'
```

Now let's add some top-level keys and values:

```
JSON.SET order $.orderNumber 6379
```

```
JSON.SET order $.orderDateTime 1682424000000
```

And a placeholder for the items ordered:

```
JSON.SET order $.items []
```

Let's check a couple of our property types:

```
JSON.TYPE order $.orderDateTime
```
```
JSON.TYPE order $.items
```

Let's add some items to our food order:

```
JSON.SET order $.items '[{"sku": "pizza001", "description": "margherita", "quantity": 4, "price": 8.50},{"sku": "pizza003", "description": "funghi", "quantity": 2, "price": 11.00},{"sku": "drinks008", "description": "Camden Hells Lager", "quantity": 16, "price": 4.00}]'
```

We can append additional items to the collection of pizza orders as follows:

```
JSON.ARRAPPEND order $.items '{"sku": "pizza002", "description": "prosciutto", "quantity": 2, "price": 13.00}'
```

We can increment individual values. Let's say we want to increase the number of drinks. Here we will increment the existing number by eight:

```
JSON.NUMINCRBY order $.items[2].quantity 8
```

```
JSON.SET order $.shipped 'false'
```
We can toggle boolean values:

```
JSON.TOGGLE order $.shipped
```

As you can see from these examples, working with JSON via the Redis CLI is efficient and intuitive once you are familiar with commands and syntax.

## Client libraries

There are many [client libraries](https://redis.io/docs/stack/json/clients/) available for different programming languages.

For .NET developers:

- https://github.com/redis/NRedisStack
- https://github.com/redis/redis-om-dotnet

## .NET Examples

In the example above we used the Redis CLI to execute RedisJSON commands. To see how we can leverage RedisJSON in applications, there are two examples provided in the following Github repository.

To get started, clone the following repository:

```
git clone https://github.com/Redislabs-Solution-Architects/redis-msft-labs.git
```

Change your directory to the following:

```
cd redis-json-example
```
Now run this command:

```
dotnet run
```

This is a simple console application to demonstrate how a JSON document can be created using .NET. In the example the NRedisStack package has been added to the project.

Notice how the methods map to Redis CLI commands that were used earlier in the lab. 

The second example is a simple web API. It's based on the example ['Todo API' from Microsoft](https://learn.microsoft.com/en-us/aspnet/core/tutorials/first-web-api?view=aspnetcore-7.0&tabs=visual-studio-code) but uses Redis and RedisJSON for persistance:

```
cd redis-api-example
```

This example shows how typical CRUD operations used by web services can be achieved using Redis OM.

Redis OM provides high-level abstractions for using Redis in .NET, making it easy to model and query your Redis domain objects.

To start the web service:

```
dotnet run
```

Now, navigate to 

```
https://localhost:7127/swagger/index.html
````


## Additional Resources

1. [Redis JSON and Search Webinar](https://github.com/Redislabs-Solution-Architects/json-search-demo)
