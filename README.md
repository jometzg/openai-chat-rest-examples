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

![alt text](openai-rest-models.png "Model deployments")

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

This will return different values each run, but here's one I prepared earlier.
```
HTTP/1.1 200 OK
Cache-Control: no-cache, must-revalidate
Content-Length: 276
Content-Type: application/json
Access-Control-Allow-Origin: *
apim-request-id: 5da2ae7a-3398-4a62-b505-2e8ab0e78dc8
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
x-ms-region: West Europe
x-content-type-options: nosniff
Openai-Model: text-davinci-003
Openai-Processing-Ms: 371.8121
X-Accel-Buffering: no
X-Ms-Client-Request-Id: 5da2ae7a-3398-4a62-b505-2e8ab0e78dc8
X-Request-Id: f1457bc1-f1e7-4871-a2b4-5203b39b2262
Date: Fri, 07 Jul 2023 14:32:25 GMT
Connection: close

{
  "id": "cmpl-7ZgnRaqHBYVzu4oRzH2Ha2vtdM3nw",
  "object": "text_completion",
  "created": 1688740345,
  "model": "text-davinci-003",
  "choices": [
    {
      "text": "\n\nthere was a",
      "index": 0,
      "finish_reason": "length",
      "logprobs": null
    }
  ],
  "usage": {
    "completion_tokens": 5,
    "prompt_tokens": 4,
    "total_tokens": 9
  }
}
```
or if I increase the number of tokens:

```
{
  "prompt": "Once upon a time",
  "max_tokens": 20
}
```

The response:
```
{
  "id": "cmpl-7ZgpXM3NxqEarGmJmfd6J2aI90F8e",
  "object": "text_completion",
  "created": 1688740475,
  "model": "text-davinci-003",
  "choices": [
    {
      "text": ", there lived three siblings, two boys and one girl, in a small village. The boys were",
      "index": 0,
      "finish_reason": "length",
      "logprobs": null
    }
  ],
  "usage": {
    "completion_tokens": 20,
    "prompt_tokens": 4,
    "total_tokens": 24
  }
}
```
