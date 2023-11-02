# Exploring how to use Microsoft Graph
This section is how to test out Microsoft Graph calls in Visual Studio Code. 

An alternative method is to the the [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer)

## Authenticating

This is using the [Resource Owner Password Credentials flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth-ropc) and is a way in which you can authenticate as a user and then get an access token.

```
# @name auth
POST https://login.microsoftonline.com/{{tenant}}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id={{clientid}}
&client_secret={{secret}}
&scope=user.read%20openid%20profile%20offline_access
&username={{email}}
&password={{password}}
&grant_type=password
```
As can be seen above, these require some settings, which can be created under a **.vscode** folder as a file **settings.json**. These allow you to remove secrets from the HTTP file. These settings also require a service principal whose clientid and secret are used below:

```
{
    "rest-client.environmentVariables": {
        "contoso": {
            "tenant": "XXXXXXX",
            "user": "johntest",
            "email": "johntest@XXXXX.com",
            "password": "XXXXXXXX",
            "clientid": "XXXXXXXX",
            "secret": "XXXXXXXX"
        
        },
        "$shared": {           
        }
    }
}
```

## The Graph "me" endpoint
```
### grab the access token for graph calls below
@test_token={{auth.response.body.access_token}}

### now graph stuff https://developer.microsoft.com/en-us/graph/graph-explorer
### now call the "me" endpoint
GET https://graph.microsoft.com/v1.0/me
Authorization: Bearer {{test_token}}
```

## Get a list of groups to which a user belongs
```
GET https://graph.microsoft.com/v1.0/me/transitiveMemberOf/microsoft.graph.group?$count=true
Authorization: Bearer {{test_token}}
```
