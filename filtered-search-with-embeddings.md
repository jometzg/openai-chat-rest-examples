# Implementing Filtered Search with Embeddings
It is often necessary to restrict access to documents, so that when a user asks an Azure OpenAI chat application, they can only search documents that apply to them.

To automate this process, the logged on user needs to be tagged in such a way that this **filtering** process is automatic.

## How this works
1. The OpenAI with data use case provides an optional filter clause to allow a search to be done but filtered.
2. So the search index needs to be built with this filter column and populated so that when the filter is applied, then only those document fagments associated with that filter get returned.
