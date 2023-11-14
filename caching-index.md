# Using Azure Search as a caching index
The usual use case of Azure Search is that to search for items or documents. In OpenAI with-data use cases, Azure Search is used to find document fragments that have been indexed from documents uploaded to blob storage. In this way a user can ground OpenAI queries on internal data.

It also may be useful to cache OpenAI responses to reduce the burden on OpenAI as a backend - this is potentially important if the quota is not quite sufficient.

The challenge is that whilst users of a chat may ask similar questions, they don't tend to ask *exactly* the same question.

As *embeddings* are a vector representation of some concept, we can use this idea to build a cache of questions and responses for a chat application.

## How Caching works
In this approach, we make use of OpenAI vector embeddings to build a vector representation of a user's question and then when the question is answered, the response stored in a cache. When later querying as cache, the question is turned into a vector and the cache is queried for the *nearest* hit - that is vector representations that are most similar to the question. It is quite likely that very similar questions will be close vector representations, so the nearest or highest score hit on the cache is likely a very similar question and this can be pulled from the cache. This is a [cache-aside](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) pattern.

## Using Azure Search as a cache
Whilst Azure Search is not strictly-speaking a cache, it may be used as one if Azure Search is already being used for indexing documents for OpenAI with-data queries.

So an index can be constructed that uses the vector representation of a question alongside the previous OpenAI response to this question.

## In Detail
The HTTP REST calls for an example of this is [here](./caching-index.http)

### Build an Index
```
POST https://{{searchinstance}}.search.windows.net/indexes?api-version=2023-07-01-Preview 
Content-Type: application/json
api-key: {{searchkey}}

{
    "name" : "{{testindex}}",
      "fields": [
        { "name": "content", "type": "Edm.String", "searchable": true,  "filterable": false, "retrievable": true },
        { "name": "last_updated","type": "Edm.String", "searchable": false, "filterable": false, "retrievable": true },
        { "name": "title",   "type": "Edm.String", "searchable": true,  "filterable": false, "retrievable": true },
        { "name": "id",      "type": "Edm.String", "searchable": false, "filterable": true,  "retrievable": true, "sortable": true, "key": true },
        { "name": "contentVector","type": "Collection(Edm.Single)", "searchable": true, "filterable": false, "retrievable": true, "sortable": false, "facetable": false, "key": false, "dimensions": 1536, "vectorSearchConfiguration": "default" }       
    ],
    "similarity": {
      "@odata.type": "#Microsoft.Azure.Search.BM25Similarity",
      "k1": null,
      "b": null
    },
  "semantic": {
    "defaultConfiguration": null,
    "configurations": [
      {
        "name": "default",
        "prioritizedFields": {
          "titleField": {
            "fieldName": "title"
          },
          "prioritizedContentFields": [
            {
              "fieldName": "content"
            }
          ],
          "prioritizedKeywordsFields": []
        }
      }
    ]
  },
  "vectorSearch": {
    "algorithmConfigurations": [
      {
        "name": "default",
        "kind": "hnsw",
        "hnswParameters": {
          "metric": "cosine",
          "m": 4,
          "efConstruction": 400,
          "efSearch": 1000
        },
        "exhaustiveKnnParameters": null
      }
    ]
  }
}
```
The above has responses in the content field and a vector index. I suspect the columns in this index can be firther reduced.

### Generate emdeddings (vectors) for a question
```
@first_question = what is the appraisal process
### generate a vector for later search
# @name firstvector
POST https://{{openaiendpoint}}.openai.azure.com/openai/deployments/{{openaiembeddingmodel}}/embeddings?api-version=2023-06-01-preview
Content-Type: application/json
api-key: {{openaikey}}

{
    "input": "{{first_question}}"
}

### get vector from response
@question_one_vector = {{firstvector.response.body.data[0].embedding}}
```
Further embeddings may be created:
```
@second_question = what is the appraisal process like
### generate a vector for later search
# @name secondvector
POST https://{{openaiendpoint}}.openai.azure.com/openai/deployments/{{openaiembeddingmodel}}/embeddings?api-version=2023-06-01-preview
Content-Type: application/json
api-key: {{openaikey}}

{
    "input": "{{second_question}}"
}

###
@question_two_vector = {{secondvector.response.body.data[0].embedding}}
```
and some completely different question:
```
@different_question = what is App Services
### generate a vector for later search
# @name threevector
POST https://{{openaiendpoint}}.openai.azure.com/openai/deployments/{{openaiembeddingmodel}}/embeddings?api-version=2023-06-01-preview
Content-Type: application/json
api-key: {{openaikey}}

{
    "input": "{{different_question}}"
}

###
@question_three_vector = {{threevector.response.body.data[0].embedding}}
```

### Upload these embeddings to the index
```
POST https://{{searchinstance}}.search.windows.net/indexes/{{testindex}}/docs/index?api-version=2023-07-01-Preview    
Content-Type: application/json   
api-key: {{searchkey}}

{
  "value": [
    {
      "@search.action": "mergeOrUpload",
      "id": "1",
      "content": "some content referring to the appraisal process",
      "title": "some title",
      "last_updated": "2023-11-14T00:00:00Z",
      "contentVector": {{question_one_vector}}
    },
     {
      "@search.action": "mergeOrUpload",
      "id": "2",
      "content": "some content refering to the appraisal process again",
      "title": "some title",
      "last_updated": "2021-10-01T00:00:00Z",
      "contentVector": {{question_two_vector}}
    },
    {
      "@search.action": "mergeOrUpload",
      "id": "3",
      "content": "some content refering to app services",
      "title": "some title",
      "last_updated": "2021-10-01T00:00:00Z",
      "contentVector": {{question_three_vector}}
    }
  ]
}
```

The above is to add/update these items in the index for testing. In a normal app, the documents will be individually added to the index as part of a cache-aside pattern in the application.

### Query the index for results
```
POST https://{{searchinstance}}.search.windows.net/indexes/{{testindex}}/docs/search?api-version=2023-10-01-Preview
Content-Type: application/json
api-key: {{searchkey}}

{
    "count": true,
    "select": "content, id",
    "vectorQueries": [
        {
            "kind": "vector",
            "vector": {{question_one_vector}},
            "exhaustive": true,
            "fields": "contentVector",
            "k": 5
        }
    ]
}
```
This then results in some JSON where the index returns with the most likely hot first:

```
{
  "@odata.context": "https://jjjdemosearch.search.windows.net/indexes('testindex')/$metadata#docs(*)",
  "@odata.count": 3,
  "value": [
    {
      "@search.score": 1.0,
      "content": "some content referring to the appraisal process",
      "id": "1"
    },
    {
      "@search.score": 0.9592271,
      "content": "some content refering to the appraisal process again",
      "id": "2"
    },
    {
      "@search.score": 0.8065599,
      "content": "some content refering to app services",
      "id": "3"
    }
  ]
}
```
As can be seen from above, the exact match his a search_score of 1.0, but a really similar question 0.959. A much less similar question has a 0.80 similarity.

In an application, it is most likely best to pick only the first response if it is above a certain threshold of similarity.
