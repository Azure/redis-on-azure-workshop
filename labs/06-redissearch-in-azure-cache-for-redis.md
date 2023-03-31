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
- Redis CLI or RedisInsight v2 (Microsoft Store [link](https://apps.microsoft.com/store/detail/redisinsight/XP8K1GHCB0F1R2))
- Redis Stack or Azure Cache for Redis (Enterprise) with the Search module enabled

## Steps

1. ...

2. ...

**Index Creation**

```
FT.CREATE idx:beers ON hash PREFIX 1 "beer:" SCHEMA name TEXT SORTABLE brewery TEXT SORTABLE breweryid NUMERIC SORTABLE category TEXT SORTABLE categoryid NUMERIC SORTABLE style TEXT SORTABLE styleid NUMERIC SORTABLE abv NUMERIC SORTABLE
```

```
FT.CREATE idx:categories ON hash PREFIX 1 "beer:" SCHEMA category TEXT PHONETIC dm:en
```

```
FT.CREATE idx:styles ON hash PREFIX 1 "beer:" SCHEMA style TEXT PHONETIC dm:en
```

```
FT.CREATE idx:breweries ON HASH PREFIX 1 "brewery:" SCHEMA name TEXT state TEXT
```

**Queries**

Get the beers that have IPA in a field:

```
FT.SEARCH idx:beers IPA
```

Get the beers that are "North American Ales":

```
FT.SEARCH idx:categories "North American Ales"
```

Get the beers with “Amreica” as the style (note the misspelling):
```
FT.SEARCH idx:styles Amreica
```
Get the beers that have a category of Lager:
```
FT.SEARCH idx:categories Lager
```
Get the beers that have a style of Lager but omit Amber:
```
FT.SEARCH idx:styles "Lager -Amber"
```
Get the beers that have an abv between 4 and 8:
```
FT.SEARCH idx:beers "@abv:[4 8]"
```
Find your favorite beer:
```
FT.Search idx:beers Pils
```
Find all the beers of your favorite category:
```
FT.Search idx:styles Pilsner
```
Find all the beers of your favorite category at your favorite brewery:
```
FT.SEARCH "idx:beers" "@category:Lager @brewery:Boulevard Brewing Company"
```
Find all of the beers where the ABV is greater than 4.5:
```
FT.SEARCH idx:beers "@abv:[(4 inf]"
```
Find all the beers that have a style of Lager but omit Amber AND have an ABV between 6 and 10:
```
FT.SEARCH idx:beers "@abv:[6 10] @style:Lager -Amber"
```

## Additional Resources

1. [Redis JSON and Search Webinar](https://github.com/Redislabs-Solution-Architects/json-search-demo)