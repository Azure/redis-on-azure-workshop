# Vector Search with NRedisStack in Azure Cache for Redis

This section guides you through utilizing vector search functionalities, offering simple examples for practice and testing. Vectors can be stored in JSON or hash documents; they are indexable, allowing you to search the dataset using the vector search feature. With Redis, you can identify semantic similarities between the database-stored data and the query vector thus enabling modern applications to implement innovative use cases leveraging semantic search. 


## Learning Objective

 In this laboratory, you will learn to use the NRedisStack client library in your C# code to:

- Generate vector embeddings using the Azure OpenAI service
- Store the vector embeddings in Redis
- Create an index 
- Perform semantic search

The proposed example will store three sentences in the database, together with their corresponding vector embeddings. You can find an introduction to using Redis as a vector database in the [docs](https://redis.io/docs/latest/develop/get-started/vector-database/) and 


## Prerequisites
- Your favorite IDE and an installation of the .NET environment to run the example proposed in this laboratory
- Azure Cache for Redis Enterprise with the RedisJSON and RediSearch modules enabled:
    - [Quickstart: Create a Redis Enterprise cache](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/quickstart-create-redis-enterprise)

        > **Please note:** When creating your Redis Enterprise cache, in the Advanced tab, select the "RedisJson" and "RediSearch" modules. This will force you to select the "Enterprise" Clustering Policy. 


## Installation

Create a .NET project from the terminal with the following command.

```
dotnet new console -o azure-vector-search -f net8.0
```

Browse into the project directory and install the [NRedisStack](https://www.nuget.org/packages/NRedisStack) and [Azure.AI.OpenAI](https://www.nuget.org/packages/Azure.AI.OpenAI) client library.

```
cd redis-test

dotnet add package NRedisStack --version 0.12.0
dotnet add package Azure.AI.OpenAI --prerelease
```

You are now ready to start this laboratory.


## Data Set <a name="dataset"></a>

In this section, you will work with the following minimal dataset of sentences.

```JSON
[
    {   
        "description": "The Commodore 64, with its iconic beige casing and 8-bit graphics, revolutionized home computing in the 1980s and remains a beloved piece of tech nostalgia today",
        "category": "computer"
    },
    {
        "description": "Stephen King's IT, with its chilling narrative and unforgettable characters, has become a quintessential read for horror enthusiasts.",
        "category": "book"
    },
    {
        "description": "The introduction of the IBM PC in 1981 set a new standard for personal computing and paved the way for the widespread adoption of PCs in businesses and homes worldwide.",
        "category": "computer"
    }
]
```

## Connect to Redis <a name="connect"></a>

Connecting to Azure Cache is done as usual, via the StackExchange.Redis client. However, you will perform indexing and searches with instances of the object `ISearchCommands`, instantiated in the following snippet.

```csharp
using NRedisStack.RedisStackCommands;
using NRedisStack.Search;
using NRedisStack.Search.Literals.Enums;
using StackExchange.Redis;
using NRedisStack;


ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost");
IDatabase db = redis.GetDatabase();
ISearchCommands ft = db.FT();

System.Console.WriteLine(db.Ping()); 
```


## Generate vector embeddings <a name="embeddings"></a>

To generate vector embeddings from text, you will use the [Azure.AI.OpenAI](https://www.nuget.org/packages/Azure.AI.OpenAI) library. 

1. Select the Microsoft Azure resource that you will be using for this example
2. Proceed to create a deployment in the [Azure OpenAI Studio](https://oai.azure.com/portal/)
3. Choose the `text-embedding-3-small` model  and give the deployment a name
4. Find the key and endpoint to access your Azure AI services API 
5. Set the corresponding`OPENAI_KEY` and the `OPENAI_ENDPOINT` environment variable

The following function accepts a sentence as input and generates a vector embedding formatted as a binary blob. Vector embeddings must be formatted as binary blobs to be stored in Redis hashes.

> Note also that the `text-embedding-3-small` embedding model generates 1536-dimensional arrays of floats. Read more about this embedding model from OpenAI's [documentation](https://platform.openai.com/docs/guides/embeddings/how-to-get-embeddings).

```csharp
private static byte[] GetEmbeddingFromAzure(string sentence){

    string endpoint = Environment.GetEnvironmentVariable("OPENAI_ENDPOINT");
    string key = Environment.GetEnvironmentVariable("OPENAI_KEY");

    Uri oaiEndpoint = new (endpoint);
    string oaiKey = key;

    AzureKeyCredential credentials = new (oaiKey);
    AzureOpenAIClient openAIClient = new (oaiEndpoint, credentials);

    EmbeddingClient embeddingClient = openAIClient.GetEmbeddingClient("text-embedding-3-small");
    Embedding returnValue = embeddingClient.GenerateEmbedding(sentence);
    Console.WriteLine(returnValue.Vector.ToArray().Length);
    // prints 1536

    // Assuming returnValue is of type OpenAI.Embeddings.Embedding
    float[] floatArray = returnValue.Vector.ToArray();
    byte[] byteArray = new byte[floatArray.Length * sizeof(float)];
    Buffer.BlockCopy(floatArray, 0, byteArray, 0, byteArray.Length);
    System.Console.WriteLine(byteArray.Length);
    // prints 6144

    return byteArray;
}   
```


## Index Creation <a name="index_creation"></a>

Now you can define the index schema, and finally create the index. The following code implements these actions:

1. It deletes any existing `vector_idx` index together with the indexed documents.
2. It defines the schema for the index. 
    - The search algorithm is `HNSW`, optimized for vector search over large amounts of vectors
    - The embedding vector stores 4-byte floats, hence you will indicate `FLOAT32`
    - The distance metric is cosine, so you will specify `COSINE`. Remember that the cosine distance is complementary to cosine similarity, this means that the closest the distance between vectors to zero, the highest the similarity.
    - Finally, the dimension of the embedding is ruled by the `text-embedding-3-small` embedding model, `1536` in this case

Note how the index is defined to track hash entries prefixed by `docs:`.
    

```csharp

try{
    ft.DropIndex("vector_idx", dd: true);
}
catch{
    Console.WriteLine("The index vector_idx does not exist");
}

var schema = new Schema()
.AddTextField(new FieldName("description", "description"))
.AddTagField(new FieldName("category", "category"))
.AddVectorField("embedding", VectorField.VectorAlgo.HNSW,
    new Dictionary<string, object>()
    {
        ["TYPE"] = "FLOAT32",
        ["DIM"] = "1536",
        ["DISTANCE_METRIC"] = "COSINE"
    }
);

SearchCommands ft = db.FT();

ft.Create(
    "vector_idx",
    new FTCreateParams().On(IndexDataType.HASH).Prefix("doc:"),
    schema);
```



## Data Loading <a name="loading"></a>

The index is now defined and created, so you can model the sentences as hashes and add the additional `embedding` field to store the vector embedding generated by OpenAI, already converted as a binary array by the function `GetEmbeddingFromAzure`.


```csharp
string sentenceComputers = "The Commodore 64, with its iconic beige casing and 8-bit graphics, revolutionized home computing in the 1980s and remains a beloved piece of tech nostalgia today.";
var hash1 = new HashEntry[] { 
    new HashEntry("description", sentenceComputers), 
    new HashEntry("category", "computer"),
    new HashEntry("embedding", GetEmbeddingFromAzure(sentenceComputers)),
};
db.HashSet("doc:1", hash1);

string sentenceRedis = "Stephen King's IT, with its chilling narrative and unforgettable characters, has become a quintessential read for horror enthusiasts.";
var hash2 = new HashEntry[] { 
    new HashEntry("description", sentenceRedis), 
    new HashEntry("category", "book"),
    new HashEntry("embedding", GetEmbeddingFromAzure(sentenceRedis)),
};
db.HashSet("doc:2", hash2);

string sentenceSpectrum = "The introduction of the IBM PC in 1981 set a new standard for personal computing and paved the way for the widespread adoption of PCs in businesses and homes worldwide.";
var hash3 = new HashEntry[] { 
    new HashEntry("description", sentenceSpectrum), 
    new HashEntry("category", "computer"),
    new HashEntry("embedding", GetEmbeddingFromAzure(sentenceSpectrum)),
};
db.HashSet("doc:3", hash3);
```


## Search Examples <a name="search_examples"></a>

The database is prepared to perform semantic searches. You will test the behavior with the sentence 

```
The Amiga 500, celebrated for its advanced graphics and sound capabilities, was a favorite among gamers and digital artists in the late 1980s and early 1990s.
```

The expectation is to get results semantically similar to this test instance.


### Syntax

[FT.SEARCH](https://redis.io/commands/ft.search/)

### Retrieve All <a name="retrieve_all"></a>

Find all documents for a given index.

#### Command

The search command requests three results (`KNN 3`) and returns the field `description` and the score, which is an indication of likelihood with the test sentence.

```csharp
string testSentence = "The Amiga 500, celebrated for its advanced graphics and sound capabilities, was a favorite among gamers and digital artists in the late 1980s and early 1990s.";

var res = ft.Search("vector_idx",
            new Query("*=>[KNN 3 @embedding $query_vec AS score]")
            .AddParam("query_vec", GetEmbeddingFromAzure(testSentence))
            .ReturnFields(new FieldName("description", "description"), new FieldName("score", "score"))
            .SetSortBy("score")
            .Dialect(2));

foreach (var doc in res.Documents) {
    Console.Write($"id: {doc.Id}, ");
    foreach (var item in doc.GetProperties()) {
        Console.Write($" {item.Value}");
    }
    Console.WriteLine();
}
```

#### Result

As mentioned, the lowest score corresponds with the highest similarity (the cosine distance is complementary to the cosine similarity), and that can be observed in the most similar sentences, both in the computer category.

```
id: doc:1,  0.482472062111 The Commodore 64, with its iconic beige casing and 8-bit graphics, revolutionized home computing in the 1980s and remains a beloved piece of tech nostalgia today.
id: doc:3,  0.664756059647 The introduction of the IBM PC in 1981 set a new standard for personal computing and paved the way for the widespread adoption of PCs in businesses and homes worldwide.
id: doc:2,  0.86530482769 Stephen King's IT, with its chilling narrative and unforgettable characters, has become a quintessential read for horror enthusiasts.
```


## Wrapping up

In this example, you have learned to model sentences using the hash data structure and perform the semantic search using an OpenAI embedding model. Remember that you can perform [hybrid](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/#pre-filter-knn-queries-hybrid-approach) and [range](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/#range-queries) searches with Redis and model your sentences using the [JSON format](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/#storing-vectors-in-json). 

## Additional Resources

1. [Connect with Redis clients ](https://redis.io/docs/latest/develop/connect/clients/)
2. [NRedisStack](https://github.com/redis/NRedisStack)
3. [Redis as a vector database quick start guide](https://redis.io/docs/latest/develop/get-started/vector-database/)
4. [Azure AI services](https://azure.microsoft.com/en-us/products/ai-services)
