# Create an Azure Search index from blob storage with embeddings
This section shows a series of REST calls that can be made to create the search index that may be used later for OpenAI queues.

This is a reverse-engineering of the steps used to generate the indexes with the OpenAI Studio chat with your own data. The solution accelerator sample application is referenced in this [public repository](https://github.com/microsoft/sample-app-aoai-chatGPT)

A Visuaal Studio Code REST Client sample for this is also [here](./create-vector-index.http)

## Overall Process
The overall process has essentially these steps:
1. upload a document to a blob container
2. create another blob container that will contain chunked and vector representation of the chunks
3. Create an intermediate data source, index and indexer to populate the secondary blob container
4. Create the main index using the secondary blob container as the source


## Create the initial blob storage as an Azure Search data source 
[Create Data Source](https://learn.microsoft.com/en-us/rest/api/searchservice/create-data-source)

```
### first create a container - haleon-new and in this put a doc
@newblobcontainer=hhh

### Create new data source
@newdatasource=hhh-datasource

POST https://{{searchinstance}}.search.windows.net/datasources?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
    "name" : "{{newdatasource}}",
    "type" : "azureblob",
    "credentials" : { "connectionString" : "DefaultEndpointsProtocol=https;AccountName={{blobaccount}};AccountKey={{blobkey}};" },
    "container" : { "name" : "{{newblobcontainer}}", "query" : "" }
}
```

## Create intermediate index

```
@newxindex=hhh-index
POST https://{{searchinstance}}.search.windows.net/indexes?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
    "name" : "{{newxindex}}",
    "fields": [
        { "name": "document_id", "type": "Edm.String", "key": true, "searchable": false },
        { "name": "content", "type": "Edm.String", "searchable": true, "filterable": false },
        { "name": "url", "type": "Edm.String", "searchable": true, "filterable": false },
        { "name": "filename", "type": "Edm.String", "searchable": true, "filterable": false },
        { "name": "metadata_storage_name", "type": "Edm.String", "searchable": false, "filterable": true, "sortable": true  },
        { "name": "metadata_storage_size", "type": "Edm.Int64", "searchable": false, "filterable": true, "sortable": true  },
        { "name": "metadata_storage_content_type", "type": "Edm.String", "searchable": false, "filterable": true, "sortable": true }       
    ]
}
```

## create a skillset that will be used to chunk and vectorise the data from the data source
```
@skillsetname=hhh-skillset
@chunkscontainer=hhh-chunks
POST https://{{searchinstance}}.search.windows.net/skillsets?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
  "name": "{{skillsetname}}",
  "skills": [
    {
      "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
      "name": "{{skillsetname}}",
      "description": null,
      "context": "/document/content",
      "uri": "https://{{openaiendpoint}}.openai.azure.com/openai/chunks?api-version=2023-03-31-preview",
      "httpMethod": "POST",
      "timeout": "PT3M",
      "batchSize": 1,
      "degreeOfParallelism": 1,
      "httpHeaders": {
           "embedding_endpoint": "https://{{openaiendpoint}}.openai.azure.com/openai/deployments/{{openaiembeddingmodel}}/embeddings?api-version=2023-03-31-preview",
           "embedding_key": "{{openaikey}}",
           "api-key": "{{openaikey}}"
      },
      "inputs": [
        {
          "name": "document_id",
          "source": "/document/document_id"
        },
        {
          "name": "filename",
          "source": "/document/filename"
        },
        {
          "name": "fieldname",
          "source": "='content'"
        },
        {
          "name": "text",
          "source": "/document/content"
        }
      ],
      "outputs": [
        {
          "name": "chunks",
          "targetName": "chunks"
        }
      ]
    }
  ],
  "cognitiveServices": null,
  "knowledgeStore": {
    "storageConnectionString": "DefaultEndpointsProtocol=https;AccountName={{blobaccount}};AccountKey={{blobkey}};EndpointSuffix=core.windows.net",
    "projections": [
      {
        "tables": [],
        "objects": [
          {
            "storageContainer": "{{chunkscontainer}}",
            "referenceKeyName": null,
            "generatedKeyName": "id",
            "source": null,
            "sourceContext": "/document/content/chunks/*",
            "inputs": [
              {
                "name": "content",
                "source": "/document/content/chunks/*/content",
                "sourceContext": null,
                "inputs": []
              },
              {
                "name": "filepath",
                "source": "/document/filename",
                "sourceContext": null,
                "inputs": []
              },
              {
                "name": "title",
                "source": "/document/content/chunks/*/title",
                "sourceContext": null,
                "inputs": []
              },
              {
                "name": "url",
                "source": "/document/url",
                "sourceContext": null,
                "inputs": []
              },
              {
                "name": "chunk_id",
                "source": "/document/content/chunks/*/chunk_id",
                "sourceContext": null,
                "inputs": []
              },
              {
                "name": "last_updated",
                "source": "/document/content/chunks/*/last_updated",
                "sourceContext": null,
                "inputs": []
              },
              {
                "name": "contentVector",
                "source": "/document/content/chunks/*/contentVector",
                "sourceContext": null,
                "inputs": []
              }
            ]
          }
        ],
        "files": []
      }
    ],
    "parameters": {
      "synthesizeGeneratedKeyName": true
    }
  },
  "encryptionKey": null
}
```


## Create Indexer
[Create Indexer](https://learn.microsoft.com/en-us/rest/api/searchservice/create-indexer)

```
@newchunksindexer=hhh-indexer
POST https://{{searchinstance}}.search.windows.net/indexers?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
  "name" : "{{newchunksindexer}}",
  "dataSourceName" : "{{newdatasource}}",
  "targetIndexName" : "{{newxindex}}",
  "skillsetName": "{{skillsetname}}",
  "parameters": {
      "batchSize": null,
      "maxFailedItems": null,
      "maxFailedItemsPerBatch": null,
      "base64EncodeKeys": null,
      "configuration": {
          "indexedFileNameExtensions" : ".txt,.md,.html,.pdf,.docx,.pptx",
          "excludedFileNameExtensions" : ".png,.jpeg",
          "dataToExtract": "contentAndMetadata",
          "parsingMode": "default"
      }
  },
  "schedule" : { },
  "fieldMappings": [
  {
      "sourceFieldName": "metadata_storage_path",
      "targetFieldName": "document_id",
      "mappingFunction": {
        "name": "base64Encode",
        "parameters": null
      }
  },
  {
      "sourceFieldName": "metadata_storage_name",
      "targetFieldName": "filename",
      "mappingFunction": null
  },
  {
      "sourceFieldName": "metadata_storage_path",
      "targetFieldName": "url",
      "mappingFunction": null
  }
  ]
}
```
the above should use the skillset populate the the intermediate blob container with document fragments and their vector representation.

You can look the container defined in "chunkscontainer" to see the document fragments.

This is the end of the first phase.

## Second Phase
Use the results of the first phase to populate the main search index. This has three steps
1. define the data source
2. Create the index
3. Create the indexer (which populates the index)

## Create a data source

```
@newdatasourcechunks=hhh-datasource-chunk
@newblobcontainerchunks=hhh-chunks
POST https://{{searchinstance}}.search.windows.net/datasources?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
    "name" : "{{newdatasourcechunks}}",
    "type" : "azureblob",
    "credentials" : { "connectionString" : "DefaultEndpointsProtocol=https;AccountName={{blobaccount}};AccountKey={{blobkey}};" },
    "container" : { "name" : "{{newblobcontainerchunks}}", "query" : "" }
}
```
This means that the "chunkscontainer" is now the source of the data for the next steps

## Create the final Index
This index is the one an application will use and has to have the correct settings to do hybrid searches

```
@newindex=hhh
POST https://{{searchinstance}}.search.windows.net/indexes?api-version=2023-07-01-Preview 
Content-Type: application/json
api-key: {{searchkey}}

{
    "name" : "{{newindex}}",
      "fields": [
    { "name": "content", "type": "Edm.String", "searchable": true,  "filterable": false, "retrievable": true },
    { "name": "filepath","type": "Edm.String", "searchable": false, "filterable": false, "retrievable": true },
    { "name": "title",   "type": "Edm.String", "searchable": true,  "filterable": false, "retrievable": true },
    { "name": "url",     "type": "Edm.String", "searchable": false, "filterable": false, "retrievable": true },
    { "name": "id",      "type": "Edm.String", "searchable": false, "filterable": true,  "retrievable": true, "sortable": true, "key": true },
    { "name": "chunk_id","type": "Edm.String", "searchable": false, "filterable": false, "retrievable": true  },
    { "name": "last_updated","type": "Edm.String", "searchable": false, "filterable": false, "retrievable": true },
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

## Create the indexer
This populates the index with the contents from the intermediate blob container (chunked and vectors)
```
@newchunkindexer=hhh-indexer-chunk
POST https://{{searchinstance}}.search.windows.net/indexers?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
  "name" : "{{newchunkindexer}}",
  "dataSourceName" : "{{newdatasourcechunks}}",
  "targetIndexName" : "{{newindex}}",
  "parameters": {
      "batchSize": null,
      "maxFailedItems": null,
      "maxFailedItemsPerBatch": null,
      "base64EncodeKeys": null,
       "configuration": {
          "parsingMode": "json",
          "indexedFileNameExtensions": ".json"
       }
  },
  "schedule" : { 
     "interval": "PT12H",
    "startTime": "2023-10-24T15:31:48.364Z"
  },
  "fieldMappings" : [ ]
}
```
The index when queried should now contain documents.

## Test Index with OpenAI Query
```
POST https://{{openaiendpoint}}.openai.azure.com/openai/deployments/{{openaichatmodel}}/extensions/chat/completions?api-version=2023-06-01-preview
api-key: {{openaikey}}
Content-Type: application/json

{
    "messages": [
        {
            "role": "user",
            "content": "what is the performance review process?"
        }
    ],
    "dataSources": [
        {
            "type": "AzureCognitiveSearch",
            "parameters": {
                "endpoint": "https://{{searchinstance}}.search.windows.net",
                "key": "{{searchkey}}",
                "indexName": "{{newindex}}",
                "inScope": false
            }
        }
    ]
}
```
If all goes well, this should return a result with a series of citations and some more interesting content:
```
      {
          "index": 1,
          "role": "assistant",
          "content": "At Contoso Electronics, performance reviews are conducted annually [doc1]. During the review, the supervisor discusses the employee's performance over the past year and provides feedback on areas for improvement. The employee also has an opportunity to discuss their goals and objectives for the upcoming year [doc1]. Performance reviews are a two-way dialogue between managers and employees, and employees are encouraged to be honest and open during the review process [doc1]. The feedback provided during performance reviews should be used as an opportunity to help employees develop and grow in their roles [doc1]. Employees receive a written summary of their performance review, which includes a rating of their performance, feedback, and goals and objectives for the upcoming year [doc1].",
          "end_turn": true
        }
```
