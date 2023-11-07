# Create Search indexes on non-public Search and Storage
It is often a security requirement that Azure Search be used in an application it has a private endpoint.

If Azure Search uses Azure blob storage as the data source for it's indexes, then it is also good practice to make the blob storage account network restricted so that it is only network visible inside an Azure virtual network.To enhance security more, managed identities are also prefered for access to the blob storage account.

The indexing process of Azure Search therefore needs to be amended to make it work with private endpoints and managed identities.

## Private endpoint concerns
There are a number of ways of building and populating Search indexes, and these often use API or REST calls. When the Search service has a private endpoint, then these scripts can no longer be executed from a remote computer or PC and so need to be executed in a VM whose network settings make the Search service network reachable. The simplest scenario is a VM in the same virtual network as the Search service (or more accurately, its private endpoint).

Ideally, this should be in customer managed GitHub Runner or DevOps build agent in the same virtual network.

For a more interactive approach, Visual Studio Code can be configured to remotely connect to a virtual machine and then API or REST requests initiated from VS Code - but these will run inside the connected VM.

