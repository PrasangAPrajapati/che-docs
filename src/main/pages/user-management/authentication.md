---
title: "Keycloak Authentication"
keywords: organizations, user management, permissions, authentication
tags: [ldap, keycloak]
sidebar: user_sidebar
permalink: authentication.html
folder: user-management
---
{% include links.html %}

## Auth Filters on master
 Che master can handle both types of signed requests - using Keycloak token or Machine token.
 Authentication tokens can be send either in a `Authorization` header or `token` query param.

 In case of Keycloak tokens, authentication filter chain contains of two filters:

- **org.eclipse.che.multiuser.keycloak.server.KeycloakAuthenticationFilter** - validates signature of the Keycloak JWT token;
- **org.eclipse.che.multiuser.keycloak.server.KeycloakEnvironmentInitalizationFilter** - extracts user data from JWT, finds or creates
     appropriate user in the Che DB, and set user principal to the session.

  If the request was made using machine token only (e.g. from ws agent) then it is only one auth filter in the chain:

- **org.eclipse.che.multiuser.machine.authentication.server.MachineLoginFilter** - finds userId given token belongs to, than retrieves user instance and sets principal to the session.

## Auth on agents
  Agents cannot be queried using Keycloak token, so there is only Machine Token option. Machine token can be passed in header or query param.

- **org.eclipse.che.multiuser.machine.authentication.agent.MachineLoginFilter** - doing basically the same as the appropriate filter on a master, the only thing that is different it's a way how agent obtains the public signature part. The public key for the signature check is placed in a machine environment, with algorithm description.
- **auth.auth.go** - the entry point for all request that is proceeding on go agents side, the logic of token verification is similar with MachineLoginFilter.

The way how Che master interacts with agents with enabled authentication mechanism is the following:
{% include image.html file="diagrams/machine_auth_flow.png" %}

## Keycloak vs Machine Token

Machine tokens were introduced to avoid passing the Keycloak tokens to the machine side (which can be potentially insecure). Another reason is that Keycloak tokens may have relatively small lifetime and require periodical renewal/refresh which is hard to manage and keep in sync with same user session tokens on clients e.t.c.

So from the implementation side, it is the simple 3-side **User** <-> **Workspace** <-> **Machine Token** mapping, and a token is injected into the machine during its startup and remains valid until the workspace will be stopped.
Like the JWT machine token contains the header, payload and signature. The structure of token and the signature are different to Keycloak and have the following view:
``` json
# Header
{
  "alg": "RS512",
  "kind": "machine_token"
}
# Payload
{
  "wsid": "workspacekrh99xjenek3h571",
  "uid": "b07e3a58-ed50-4a6e-be17-fcf49ff8b242",
  "uname": "john",
  "jti": "06c73349-2242-45f8-a94c-722e081bb6fd"
}
# Signature
{
  "value": "RSASHA512(base64UrlEncode(header) + . +  base64UrlEncode(payload))"
}

```
The algorithm that is used for signing machine tokens is `SHA-512` and it's not configurable for now. Also, there is no public service that distributes the public part of the key pair with which the token was signed. But in each machine, there must be environment variables that contains key value. Environment properties description:

Contains information about the algorithm which the token was signed<br>
`CHE_MACHINE_AUTH_SIGNATURE__ALGORITHM`<br>
Contains public key value encoded in Base64<br>
`CHE_MACHINE_AUTH_SIGNATURE__PUBLIC__KEY`


## Obtaining Keycloak endpoints

There is separate REST service to obtain main Keycloak endpoints for the current installation. It helps to clients to find out URLs for basic operations like login/logout, profile reading, realm name etc. This service is not secured with any authentication and accessible for any client. Service URL is: `http://<wsmaster_host>:<port>/api/keycloak/settings`
Example output:
```
{
  "che.keycloak.token.endpoint": "http://172.19.20.9:5050/auth/realms/che/protocol/openid-connect/token",
  "che.keycloak.profile.endpoint": "http://172.19.20.9:5050/auth/realms/che/account",
  "che.keycloak.client_id": "che-public",
  "che.keycloak.auth_server_url": "http://172.19.20.9:5050/auth",
  "che.keycloak.password.endpoint": "http://172.19.20.9:5050/auth/realms/che/account/password",
  "che.keycloak.logout.endpoint": "http://172.19.20.9:5050/auth/realms/che/protocol/openid-connect/logout",
  "che.keycloak.realm": "che"
}
```

## Obtaining Token From Keycloak

The simplest way to obtain Keycloak auth token, is to perform request to the token endpoint with username and password credentials. This request can be schematically described as following cURL request:

```bash
curl
    --data "grant_type=password&client_id=<client_name>&username=<username>&password=<password>"
     http://<keyckloak_host>:5050/auth/realms/<realm_name>/protocol/openid-connect/token
```

 Since the two main Che clients (IDE and Dashboard) utilizes native Keycloak js library, they're using a customized Keycloak login page and somewhat more complicated authentication mechanism using `grant_type=authorization_code`. It's like a two step authentication, when first step is login and obtaining authorization code, and second step is obtaining token using this code.

 Example of correct token response:

 ```json
 {
   "access_token":"eyJhb...<rest of JWT token here>",
   "expires_in":300,
   "refresh_expires_in":1800,
   "refresh_token":"Nj0C...<rest of refresh token here>",
   "token_type":"bearer",
   "not-before-policy":0,
   "session_state":"14de1b98-8065-43e1-9536-43e7472250c9"
 }
 ```

## Using Swagger or REST Clients

User's Keycloak token is used to execute queries to secured API on his behalf through REST clients. A valid token must be attached as request header or query parameter - `?token=$token`. Che Swagger can be accessed from `http://che_host:8080/swagger`. A user must be signed-in through Keycloak so that access token is included in request headers.

By default, swagger loads `swagger.json` from [Master][che-master].

To work with WS Agent, a URL to its `swagger.json` should be provided. It can be retrieved from [Workspace Runtime](workspace-data-model.html#runtime), by getting URL to [WS Agent server][servers] endpoint and adding `api/docs/swagger.json` to it. Also, to authenticate on WS Agent API, user must include Machine Token, which can be found in Workspace Runtime as well.

To use Swagger for a workspace agent, user must do following steps:

* get workspace object with runtimes fetched (using `/api/workspace/<workspace_key>` service)
* get WS agent API endpoint URL, and add a path to its `swagger.json` (e.g. `http://<che_host>:<machine_port>/api/docs/swagger.json` for Docker or `http://<ws-agent-route>/api/docs/swagger.json` for OpenShift ). Put it in the upper bar `URL` field:

```json
"wsagent/http": {
  "url": "http://172.19.20.180:32777/api",
  "attributes": {},
  "status": "RUNNING"
}
```
* get machine token from `runtime.machineToken` field, and put it in the upper bar `token` field

```
"machineToken": "eyJhbGciOiJSUzUxMiIsImtpbmQiOiJtYWNoaW5lX3Rva2VuIn0.eyJ3c2lkIjoid29ya3NwYWNlMzEiLCJ1aWQiOiJ1c2VyMTMiLCJ1bmFtZSI6InRlc3RVc2VyIiwianRpIjoiOTAwYTUwNWYtYWY4ZS00MWQxLWFhYzktMTFkOGI5OTA5Y2QxIn0.UwU7NDzqnHxTr4vu8UqjZ7-cjIfQBY4gP70Nqxkwfx8EsPfZMpoHGPt8bfqLWVWkpp3OacQVaswAOMOG9Uc9FtLnQWnup_6vvyMo6gchZ1lTZFJMVHIw9RnSJAGFl98adWe3NqE_DdM02PyHb23MoHqE_xd8z3eFhngyaMImhc4",
```

* click Explore to load Swagger for WS Agent

{% include image.html file="devel/swagger.png" %}