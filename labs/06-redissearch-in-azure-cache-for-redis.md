# RediSearch in Azure Cache for Redis

To store data in Redis, we first define a key and then set the value. For example, to add a Redis Hash: 

`hset participant-id:0001 name satya`

If we know the key ahead of time, this works great for retrieving a value. 

However, what about scenarios where there's data in the cache that needs to be searched or aggregated? For example, a product, stores or transactions by a particular vendor? 

Typically this is where a key/value model starts to show its limitations. To search or carry out processing, large transfers of data may be required by a client application.

RediSearch adds search capabilities to Redis and includes many other powerful features, and it does so right where your data lives. This means you can process much larger volumes of data and at greater speed.

## Learning Objective
- Learn which data structures can be used with RediSearch.
- Explore an example data set.
- Create a RediSearch index.
- Perform queries using RediSearch commands.

## Prerequisites
...

## Steps

1. ...

2. ...

## Additional Resources

1. [Redis JSON and Search Webinar](https://github.com/Redislabs-Solution-Architects/json-search-demo)