---
title: API Integration
layout: home
nav_order: 10
---


# API Integration
This section describes the API integration to Signaturgruppen Privilege Management in more detail. 

> [Old PDF documentation is found here](https://www.signaturgruppen.dk/download/broker/docs/Nets%20eID%20Broker%20Privilege%20Management.pdf)

## Swagger definition
The API definition is published through the Signaturgruppen Privilege Management Swagger API definition found at

PP environment:
```
https://pp.netseidbroker.dk/privileges-api/swagger/index.html
```

Production environment:
```
https://netseidbroker.dk/privileges-api/swagger/index.html
```

> Note that all references are for the PP environment in the rest of the documentation.

## Basic lookup using Signaturgruppen Broker access token

For reference, see the [Signaturgruppen Broker documentation](https://signaturgruppen-a-s.github.io/signaturgruppen-broker-documentation/).

The most basic integration and way to lookup privileges for a logged in user, is to utilize the access token received as part of the OpenID Connect login utilized for any Signaturgruppen Broker authentication flow. 

In order to enable the returned access tokens for privilege lookups, the **privileges** scope value must be added to the Signaturgruppen Broker authentication request.

Example scopes for MitID Erhverv request
```
scope=openid nemlogin privileges
```

Upon successful authentication, the access token received on behalf of the end-user is used as a bearer token for the Privilege API runtime endpont

```
Authorization: Bearer {access token}
GET https://pp.netseidbroker.dk/privileges-api/api/v1/identity/privileges
```
> **NOTE** This is the only Privilege API that accepts an end-user **access token** received from a NeB OIDC flow

### Example with access token from brokerdemo
The online demo login site for Nets eID Broker is found at https://brokerdemo-pp.signaturgruppen.dk/ and allows for setting up many of the possible authentication flows available throught the Nets eID Broker platform. 

After a successful login, the user is sent to a page that lists all claims received by the service provider (the demo site) and further lists the various tokens received from the OpenID Connect flow. Here you can see and copy the received access token for the flow and use this to test out the basic runtime integration to the privilege API using access tokens. 

In the following a login was made through the demo site with a MitID test user and then the privileges owned by Signaturgruppen, as the demo site is a Signaturgruppen site, assigned to this user is returned.  

> **NOTE** Access tokens should never be sent to browsers or directly to end-users in production

```
MitID user: privilege_demo_user_1
MitID UUID: 06087080-dab1-4015-bf88-a4b4d6486537
Access token: copied from login result from demo site
```

To lookup the Signaturgruppen owned privileges assigned to the MitID user, the following Privileges API endpoint is called

```
Authorization: Bearer {access token}
GET https://pp.netseidbroker.dk/privileges-api/api/v1/identity/privileges
```

Producing the following output (at that moment):


```
{
    "identity": {
        "idpIdentityId": "06087080-dab1-4015-bf88-a4b4d6486537",
        "idp": "mitid"
    },
    "clientInfo": {
        "clientId": "84e11a16-d93b-48a6-8dad-bf5ef26f31be",
        "organizationTin": "DK29915938"
    },
    "organizationScopes": [
        {
            "organizationTin": "DK29915938",
            "privileges": [
                {
                    "id": "cb0ae2a1-0aa1-4bac-fae5-08db4a8295cd",
                    "name": "Demo Internal Admin",
                    "updated": "2023-05-03T20:29:59.3461406+00:00"
                }
            ]
        },
        {
            "organizationTin": "DK00000002",
            "privileges": [
                {
                    "id": "ebb63417-1c05-4799-fae4-08db4a8295cd",
                    "name": "Demo Accountant",
                    "updated": "2023-05-03T20:28:48.4801303+00:00"
                }
            ]
        }
    ]
}
```

This shows, that the MitID user has been assigned the Signaturgruppen privilege of "Demo Accountant" on behalf of the organization DK00000002 and that the user also has been assigned the Signaturgruppen privilege "Demo Internal Admin" on behalf of Signaturgruppen.

> **NOTE** The user might have been assigned privileges on behalf of more than one organization, as in the example above.


## Lookup using the API integration
For all other scenarios on the API an API Client and a service token is required. Service tokens are an OAuth access token issued directly to an API client instead of an end-user, otherwise technically the same thing.

If the access token runtime API is not an option or if the service provider wants to be able to lookup runtime privileges without first having to authenticate the end-user, then the following Privilege API for runtime lookups is available using a **service token** (i.e. not an end-user access token):

This requires a service token retrieved by calling the NeB Token endpoint (PP: https://pp.netseidbroker.dk/op/connect/token) and using the resulting service token as bearer token. 

> The API client invoking the API must have been assigned the Signaturgruppen privilege **AssignIdentityPrivilegesRead**

> The **privilege_api** scope must be requested in the Client Credentials request for service tokens

**Header**

| Key | Value |
|---|---|
| Content-Type | application/json |


**Body**

| Key | Value |
|---|---|
| grant_type | client_credentials |
| client_id | fill in client_id |
| client_secret | fill in client_secret |
| scope | privilege_api |


**Returns:**

`{
    "access_token": "eyJh...removed... ",
    "expires_in": 3600,
    "token_type": "Bearer",
    "scope": "privilege_api"
}`

The service token **(the access_token parameter)** can then be used agaist the Privilege API as a Bearer token.




### Full example
In the following it is demonstrated how to retrieve a service token used to invoke almost all Privilege API endpoints on behalf of your service integration and utilize this to lookup the privileges for a specific MitID identity. This is done on behalf of the test organization DK00000002 for which an API client is provided here in the example. 

The test organization DK00000002 has created a public privilege called "Revisor" and two internal privileges: "Supporter" and "Administrator".

```
MitID user: privilege_demo_user_1
MitID UUID: 06087080-dab1-4015-bf88-a4b4d6486537
Access token: copied from login result from demo site
```

The DK00000002 has an API client for their API integration with the following credentials:

```
Test organization: DK00000002
clientID: 6075a779-713a-4d6f-9a48-ccce85c4bcbc
clientSecret: d8qXriaUHgH2BpikD/jsEChKDKUKCKq5aAnAKqnwlE10OCEr8l7skptyCc7FZOGURorMNTAFuFZmyhAxYDHqDQ==
```

First step is to retrieve a working service token usable towards the Privilege API. This is done with a standard OAuth Client Credentials flow asking for the "privilege_api" scope.

```
https://pp.netseidbroker.dk/op/connect/token
Content-Type: application/x-www-form-urlencoded

grant_type:client_credentials
scope:privilege_api
client_id:6075a779-713a-4d6f-9a48-ccce85c4bcbc
client_secret:d8qXriaUHgH2BpikD/jsEChKDKUKCKq5aAnAKqnwlE10OCEr8l7skptyCc7FZOGURorMNTAFuFZmyhAxYDHqDQ==

```

This results in a response that includes the requested access token (service token)

```
{
    "access_token": "eyJhbGciOiJSU...r3LZqmA",
    "expires_in": 3600,
    "token_type": "Bearer",
    "scope": "privilege_api"
}
```

Then the following Privilege API endpoint is called 

```
Authorization: Bearer {access token}
https://pp.netseidbroker.dk/privileges-api/api/v1/organizations/DK00000002/assignedIdentityPrivileges/mitid/06087080-dab1-4015-bf88-a4b4d6486537/runtime

```
> The **{idp}** parameter (mitid) is the idp claim from a NeB login and also matches the defined idp claim values from the NeB technical documentation.  

> The **{IdpIdentityId}** (06087080-dab1-4015-bf88-a4b4d6486537) parameter is the unique and global identifer for the identity for the IDP, for instance a UUID for both MitID and MitID Erhverv. This is the same values used for assigning privileges using the API.

The call results in the following JSON (at that time)

```
{
    "identity": {
        "idpIdentityId": "06087080-dab1-4015-bf88-a4b4d6486537",
        "idp": "mitid"
    },
    "clientInfo": {
        "clientId": "afa009de-d523-437c-8cc0-9b4208d7f2c6",
        "organizationTin": "DK00000002"
    },
    "organizationScopes": [
        {
            "organizationTin": "DK29915938",
            "privileges": [
                {
                    "id": "ec94f8bc-f38d-4abc-76c8-08dad14a2dfc",
                    "name": "Revisor",
                    "updated": "2022-11-28T14:09:35.5206677+00:00"
                }
            ]
        },
        {
            "organizationTin": "DK00000002",
            "privileges": [
                {
                    "id": "db4fbc0e-9d04-4c3c-5bd6-08dad14a0997",
                    "name": "Supporter",
                    "updated": "2022-11-28T14:08:34.4581783+00:00"
                },
                {
                    "id": "24c7a2ab-492d-4d4c-76c9-08dad14a2dfc",
                    "name": "Administrator",
                    "updated": "2022-11-28T20:36:39.2186688+00:00"
                }
            ]
        }
    ]
}
```
Showing that the user has been assigned both "Supporter" and "Administrator" by DK00000002 and has been assigned "Revisor" by Signaturgruppen.
