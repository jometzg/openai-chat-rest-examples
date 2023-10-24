# Create an Azure Search index from blob storage
This section shows a series of REST calls that can be made to create the search index that may be used later for OpenAI queues.

This involves a number of steps outlined in the [Indexer operations Azure Cognitive Search REST API](https://learn.microsoft.com/en-us/rest/api/searchservice/indexer-operations) documentation.

A more comprehensive sample using document chunking and embeddings is create-vector-index.http in this repo.


## Create a blob storage data source 
[Create Data Source](https://learn.microsoft.com/en-us/rest/api/searchservice/create-data-source)

```
### create indexer data source
@blobaccount = YOUR_STORAGE_ACCOUNT_NAME
@blobkey= XXXXXXX
@blobcontainer= documents
@indexerdatasource = my-blob-datasource

POST https://{{searchservice}}.search.windows.net/datasources?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
    "name" : "{{indexerdatasource}}",
    "type" : "azureblob",
    "credentials" : { "connectionString" : "DefaultEndpointsProtocol=https;AccountName={{blobaccount}};AccountKey={{blobkey}};" },
    "container" : { "name" : "{{blobcontainer}}", "query" : "" }
}
```

## Create index

```
### create indexer definition
@newindex = my-search-index

POST https://{{searchservice}}.search.windows.net/indexes?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
    "name" : "{{newindex}}",
    "fields": [
        { "name": "ID", "type": "Edm.String", "key": true, "searchable": false },
        { "name": "content", "type": "Edm.String", "searchable": true, "filterable": false },
        { "name": "metadata_storage_name", "type": "Edm.String", "searchable": false, "filterable": true, "sortable": true  },
        { "name": "metadata_storage_size", "type": "Edm.Int64", "searchable": false, "filterable": true, "sortable": true  },
        { "name": "metadata_storage_content_type", "type": "Edm.String", "searchable": false, "filterable": true, "sortable": true }       
    ]
}
```

## Create Indexer
[Create Indexer](https://learn.microsoft.com/en-us/rest/api/searchservice/create-indexer)

```
### create indexer
@newindexer = my-blob-indexer

POST https://{{searchservice}}.search.windows.net/indexers?api-version=2020-06-30
Content-Type: application/json
api-key: {{searchkey}}

{
  "name" : "{{newindexer}}",
  "dataSourceName" : "{{indexerdatasource}}",
  "targetIndexName" : "{{newindex}}",
  "parameters": {
      "batchSize": null,
      "maxFailedItems": null,
      "maxFailedItemsPerBatch": null,
      "base64EncodeKeys": null,
      "configuration": {
          "indexedFileNameExtensions" : ".pdf,.docx",
          "excludedFileNameExtensions" : ".png,.jpeg",
          "dataToExtract": "contentAndMetadata",
          "parsingMode": "default"
      }
  },
  "schedule" : { },
  "fieldMappings" : [ ]
}
```
the above is a process that may take some time depending on the number of documents in the blob container.

## Check status of indexer
[Get Indexer Status](https://learn.microsoft.com/en-us/rest/api/searchservice/get-indexer-status)

```
### get indexer status
GET https://{{searchservice}}.search.windows.net/indexers/{{newindexer}}/status?api-version=2020-06-30
Content-Type: application/json  
api-key: {{searchkey}}
```
When indexing is complete, it can be tested using an OpenAI call - referencing this new index.

## Test Index with OpenAI Query
```
### chat with private data - over the new index, created above
POST https://{{deployment}}.openai.azure.com/openai/deployments/{{model}}/extensions/chat/completions?api-version=2023-06-01-preview
api-key: {{api-key}}
Content-Type: application/json

{
    "messages": [
        {
            "role": "system",
            "content": "You are a helpful AI assistant"
        },
        {
            "role": "user",
            "content": "what documents are there?"
        }
    ],
    "dataSources": [
        {
            "type": "AzureCognitiveSearch",
            "parameters": {
                "endpoint": "https://{{searchservice}}.search.windows.net",
                "key": "{{searchkey}}",
                "indexName": "{{newindex}}"
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
          "content": "I have found 3 documents related to your query. Would you like me to provide you with more information about each document?",
          "end_turn": true
}
```
