# RedisJSON in Azure Cache for Redis

JSON (JavaScript Object Notation) is an open standard file and data interchange format. If you've worked with any modern web application or API, you've probably used JSON.

JSON is supported by most programming languages, and typically native objects and arrays are encoded as JSON (serialized), before being sent over the network or written to a file. Parsing JSON, or deserialization, is the opposite of this process.

Storing JSON in Azure Cache for Redis let's you work with JSON values directly, without the need for multiple operations to retrieve, update or access values within the data. When working with JSON in your cache, there's no need to serialize and deserialize data.

The RedisJSON module provides JSON support for Redis. RedisJSON lets you store, update, and retrieve JSON values in a Redis database, similar to any other Redis data type.

Azure Cache for Redis Enterprise includes RedisJSON. Once enabled, your cache becomes a powerful, superfast in-memory JSON document store.

## Learning Objective
- How to use Redis commands for RedisJSON
- RedisJSON using RedisOM in a .NET application

## Prerequisites
- Redis CLI
- Redis Stack or Azure Cache for Redis (Enterprise) with the JSON module enabled
- .NET 6

## Steps

Lets imagine a scenario where we are working for an online food order company. Redis and Microsoft are running a developer meetup and have decided to order in some food and refreshments for the guests.

Let's consider what a typical JSON object might look like that contains the data for the order:

**JSON object here**

We could do this using one command, but let's build up our object incrementally. Here we start with an empty object:

```
JSON.SET order $ '{}'
```

Using `JSON.SET`, you can choose to overwrite data or work with values using JSONPath (hence the `$` - this denotes the root of the JSON object). For example, let's overwrite this inital empty object:

```json
JSON.SET order $ '{"deliveryAddress": "Microsoft UK, Thames Valley Park, Reading", "orderTotal": 0.00, "paid": false}'
```

Now let's add some top-level keys and values:

```json
JSON.SET order $.orderNumber 6379
```

```json
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

Let's add our pizzas!

```json
JSON.SET order $.items '[{"sku": "pizz-001", "description": "margherita", "quantity": 4, "price": 8.50},{"sku": "pizz-003", "description": "funghi", "quantity": 2, "price": 11.00},{"sku": "dri-008", "description": "Camden Hells Lager", "quantity": 16}]'
```

We can append additional items to the collection of pizza orders as follows:

```json
JSON.ARRAPPEND order $.items '{"sku": "pizz-002", "description": "prosciutto", "quantity": 2, "price": 13.00}'
```

We can increment individual values. Let's say we want to increase the number of drinks. Here we will increment the existing number by eight:

```
JSON.NUMINCRBY order $.items[2].quantity 8
```

```json
JSON.SET order $.shipped 'false'
```

```
JSON.TOGGLE order $.shipped
```

1. ...

2. ...

## Additional Resources

1. [Redis JSON and Search Webinar](https://github.com/Redislabs-Solution-Architects/json-search-demo)