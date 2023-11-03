# Implementing Filtered Search with Embeddings
It is often necessary to restrict access to documents, so that when a user asks an Azure OpenAI chat application, they can only search documents that apply to them.

To automate this process, the logged on user needs to be tagged in such a way that this **filtering** process is automatic.

One way to do this is to associate the user with An Azure AD Group (Entra ID group) and this association only allows documents that "belong" to this group.

## How this works
1. The OpenAI with data use case provides an optional [filter](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference) clause to allow a search to be done but filtered.
4. The search index needs to be built with this filter column and populated so that when the filter is applied, then only those document fragments associated with that filter get returned. See [document-level access control](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data#document-level-access-control)
5. The logged on user's identity can be inspected by the application to list the AD group IDs to which they belong. The sample application already has this code in *app.py* with the function *generateFilterString*. See the [sample application code](https://github.com/microsoft/sample-app-aoai-chatGPT/blob/main/app.py)
6. These can then be added to the filter clause, so that correctly tagged documents will only be searched
7. At the document level, one approach to this is to set custom meta data onto the documents in blob storage, so that the indexing process picks up this meta-data and then applies this to the indexes.
8. If the document is in Azure blob storage, a meta data tag **group_ids** can be added that contains an array of the AD group IDs that are allowed to view the indexes created from this.
9. This meta data is then picked in the indexing process and applied to the main index
10. In the application, if enabled, the user's AD group ID memberships are queried to then automatically construct a filter onto the data.
11. When ssearching this then only queries document fragments that meeet the filter criteria.

## Setting up the indexes
See the [REST example](./filtered-index.http)

## Create AD Groups

![alt text](./set-ad-groups.png "set AD groups")

## Add User membership to these groups
![alt text](./set-ad-group-membership.png "set AD groups")

## Set blob meta data
See the guide on [document-level access control](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data#document-level-access-control) 

![alt text](./set-blob-meta-data.png "Set blob meta data")

## Resultant document fragments
We can see below that the main index the application uses has a new field *group_ids* which contains an array of Azure AD group IDs. It is this field that the application can use to filter the results.

![alt text](./document-fragments-in-index.png "Document fragments")


## Building the filters in the application
This code is present in the sample application:

![alt text](./build-filter-criteria.png "Document fragments")


