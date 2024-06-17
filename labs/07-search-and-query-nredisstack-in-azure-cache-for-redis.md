# Search and Query with NRedisStack in Azure Cache for Redis

In the previous laboratories, you have learned to store data in Redis using the JSON data structure from the `redis-cli` command line client. This laboratory will teach you how to store, index and search the same dataset from your C#/.NET application using the [NRedisStack](https://github.com/redis/NRedisStack) client library.
 
The NRedisStack client library for C#/.NET applications introduces native support for Redis capabilities to the C# environment. You can work with secondary indexing, JSON support, vector search, time series and probabilistic data structures. Developed on the foundation of Microsoft's StackExchange.Redis client library, it offers commands for handling data structures.


## Learning Objective

 In this laboratory, you will learn to use the NRedisStack client library in your C# code to:

- Insert JSON documents
- Create an index on the desired fields
- Perform search operations


## Prerequisites
- Your favorite IDE and an installation of the .NET environment to run the example proposed in this laboratory
- Azure Cache for Redis Enterprise with the RedisJSON and RediSearch modules enabled:
    - [Quickstart: Create a Redis Enterprise cache](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/quickstart-create-redis-enterprise)

        > **Please note:** When creating your Redis Enterprise cache, in the Advanced tab, select the "RedisJson" and "RediSearch" modules. This will force you to select the "Enterprise" Clustering Policy. 

## Installation

Create a .NET project from the terminal with the following command.

```
dotnet new console -o redis-test -f net8.0
```

Browse into the project directory and install the [NRedisStack](https://www.nuget.org/packages/NRedisStack) client library.

```
cd redis-test

dotnet add package NRedisStack --version 0.12.0
```

You are now ready to start this laboratory.


## Data Set <a name="dataset"></a>

In this section, you will work with the following dataset. Note that the `season` is modeled as a multi-category attribute, and the location is expressed by the longitude and latitude coordinates, in this order. It is important to preserve this order, which is considered at indexing time.

```JSON
[
    {   
        "id": 15970,
        "gender": "Men",
        "season":["Fall", "Winter"],
        "description": "Turtle Check Men Navy Blue Shirt",
        "price": 34.95,
        "city": "Boston",
        "location": "-71.057083, 42.361145"
    },
    {
        "id": 59263,
        "gender": "Women",
        "season": ["Fall", "Winter", "Spring", "Summer"],
        "description": "Titan Women Silver Watch",
        "price": 129.99,
        "city": "Dallas",
        "location": "-96.808891, 32.779167"
    },
    {
        "id": 46885,
        "gender": "Boys",
        "season": ["Fall"],
        "description": "Ben 10 Boys Navy Blue Slippers",
        "price": 45.99,
        "city": "Denver",
        "location": "-104.991531, 39.742043"
    }
]
```

## Connect to Redis <a name="connect"></a>

Connecting to Azure Cache is done as usual, via the StackExchange.Redis client. However, you will operate with JSON documents, indexing and searches with instances of the objects `ISearchCommands` and `IJsonCommands`, instantiated in the following snippet.

```csharp
using NRedisStack.RedisStackCommands;
using NRedisStack.Search;
using NRedisStack.Search.Literals.Enums;
using StackExchange.Redis;
using NRedisStack;


ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost");
IDatabase db = redis.GetDatabase();
ISearchCommands ft = db.FT();
IJsonCommands json = db.JSON();

System.Console.WriteLine(db.Ping()); 
```


## Index Creation <a name="index_creation"></a>

Now you can define the index schema, and finally create the index.

```csharp
var schema = new Schema()
    .AddNumericField(new FieldName("$.id", "id"))
    .AddTagField(new FieldName("$.gender", "gender"))
    .AddTagField(new FieldName("$.season.*", "season"))
    .AddTextField(new FieldName("$.description", "description"))
    .AddNumericField(new FieldName("$.price", "price"))
    .AddTextField(new FieldName("$.city", "city"))
    .AddGeoField(new FieldName("$.location", "location"));

ft.Create(
    "idx1",
    new FTCreateParams().On(IndexDataType.JSON).Prefix("product:"),
    schema);
```

You can review the specification for the index just created using the `FT.INFO` command, executed from `redis-cli`.


## Data Loading <a name="loading"></a>

You will now create the JSON objects as follows, and import them.

```csharp
var product1 = new {
    id = 15970,
    gender = "Men",
    season = new string[] {"Fall", "Winter"},
    description = "Turtle Check Men Navy Blue Shirt",
    price = 34.95,
    city = "Boston",
    location = "-71.057083, 42.361145"
};

var product2 = new {
    id = 59263,
    gender = "Women",
    season = new string[] {"Fall", "Winter", "Spring", "Summer"},
    description = "Titan Women Silver Watch",
    price = 129.99,
    city = "Dallas",
    location = "-96.808891, 32.779167"
};

var product3 = new {
    id = 46885,
    gender = "Boys",
    season = new string[] {"Fall"},
    description = "Ben 10 Boys Navy Blue Slippers",
    price = 45.99,
    city = "Denver",
    location = "-104.991531, 39.742043"
};

json.Set(key: "product:15970", path: "$", product1);
json.Set(key: "product:59263", path: "$", product2);
json.Set(key: "product:46885", path: "$", product3);
```

## Search Examples <a name="search_examples"></a>

You can finally perform the search operation, and add filters as learned in the previous laboratory.

### Syntax

[FT.SEARCH](https://redis.io/commands/ft.search/)

### Retrieve All <a name="retrieve_all"></a>

Find all documents for a given index.

#### Command

```csharp
SearchResult SearchResult = ft.Search("idx1", new Query("*"));
Console.WriteLine(SearchResult.TotalResults);
Console.WriteLine(string.Join("\n", SearchResult.ToJson()));
```
#### Result

```
3
{"id":15970,"gender":"Men","season":["Fall","Winter"],"description":"Turtle Check Men Navy Blue Shirt","price":34.95,"city":"Boston","location":"-71.057083, 42.361145"}
{"id":59263,"gender":"Women","season":["Fall","Winter","Spring","Summer"],"description":"Titan Women Silver Watch","price":129.99,"city":"Dallas","location":"-96.808891, 32.779167"}
{"id":46885,"gender":"Boys","season":["Fall"],"description":"Ben 10 Boys Navy Blue Slippers","price":45.99,"city":"Denver","location":"-104.991531, 39.742043"}
```

### Single Term Text <a name="single_term"></a>

Find all documents with a given word in the description text field.

#### Command

```csharp
SearchResult = ft.Search("idx1", new Query("@description:Slippers"));
Console.WriteLine(string.Join("\n", SearchResult.ToJson()));
```

#### Result

```bash
{"id":46885,"gender":"Boys","season":["Fall"],"description":"Ben 10 Boys Navy Blue Slippers","price":45.99,"city":"Denver","location":"-104.991531, 39.742043"}
```

### Return selected values <a name="selected_values"></a>

Find the documents with the price in the specified range and return the `id` and `price` fields.

#### Command

```
SearchResult = ft.Search("idx1", new Query("@price:[30 40]").ReturnFields(new FieldName("$.id", "id"), new FieldName("$.price", "price")));
Console.WriteLine(string.Join(", ", SearchResult.Documents.Select(d => d["id"] + " " + d["price"])));
```

#### Result

```
15970 34.95
```

## Additional Resources

1. [Connect with Redis clients ](https://redis.io/docs/latest/develop/connect/clients/)
2. [NRedisStack](https://github.com/redis/NRedisStack)
___
[Next Lab](./08-vector-search-nredisstack-in-azure-cache-for-redis.md)
