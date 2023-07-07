# Azure OpenAI Chat Samples using REST calls
Azure OpenAI has the ability to assist in a number of AI/cognitive services, so it is important to understand how this works.

There are a number of samples and SDKs, but I often prefer getting down to the *bare metal* of HTTP REST calls. This repo shows how to do a few use cases directly as REST requests. These are:
1. Completions
2. Basic chat
3. Chat with embeddings - that is, over your own data.

I personally use the [REST client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)  in Visual Studio code as this can use parameters and chain requests. Check it out.

## Prerequisites
In order to do any of these demos, you need to have deployed an Azure OpenAI service into your Azure subscription. This service is being rationed and so you may need to apply for it.

In addition, you also need to deploy some models into that instance. Note that the region you choose for OpenAI is important as not all models are available in all regions. I personally use *West Europe* as this at the time of writing has good model availability. The models needed will depend on the uses cases below.



## Completions
This is a really simple use case where you send a piece of test and it will attempt to "complete" what you started. A nice simple service.

```
### some variables
@deployment = your_openai_name
@api-key = XXXXXXXXXXXXXXXXXX
@model = text-davinci-003

### call OpenAI REST interface - completions
POST https://{{deployment}}.openai.azure.com/openai/deployments/{{model}}/completions?api-version=2023-05-15
api-key: {{api-key}}
Content-Type: application/json

{
  "prompt": "Once upon a time",
  "max_tokens": 5
}
```
