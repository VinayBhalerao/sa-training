# 3scale & RHSSO OAuth (Client Credentials Grant Flow)
![flow](https://d26dzxoao6i3hh.cloudfront.net/items/0c202I421N3y0W0l1y25/3scale%20-%20RH%20SSO%20OAuth%20Client%20Credentials%20Grant%20%20(1).png)

This guide walks you through:

* Installing rhsso
* Installing APIcast (docker) using RHSSO_ENDPOINT environment variable
* Creating a new service in 3scale
* Creating a new Application in 3scale
* Creating a new client in RHSSO and copy client creds from 3scale
* Flow testing

## Installing RHSSO

Refer Kavitha's [instructions](https://github.com/kasriniv/openshiftv3-ops-workshop/blob/master/3scale/RHSSOInstall.md) on how to install RHSSO

## Installing RHSSO (Optional - if rhsso doesn't work)

In this example, keylcoak will be exposed on 3000
`docker run -d -e RHSSO_USER=admin -e RHSSO_PASSWORD=admin -p 3000:8080  jboss/keycloak`


## Installing APIcast
APIcast is exposed on 8080
```docker run --name apicast -d -p 8080:8080 -e THREESCALE_PORTAL_ENDPOINT=https://{ACCESS-TOKEN}@{ACCOUNT}-admin.3scale.net -e RHSSO_ENDPOINT=http://{rhsso HOST}:{rhsso PORT}/auth/realms/{REALM} registry.access.redhat.com/3scale-amp20/apicast-gateway:1.0```


## Creating a Service in 3scale
![Create](https://d26dzxoao6i3hh.cloudfront.net/items/3i440Q0e2W0x1b3N3H3O/%5Ba1c0d312b05a4be2780985ecece05592%5D_Screen%2520Shot%25202017-07-21%2520at%252011.15.13.png?v=33082937)

**Set up the service as follows:** 

 * Enter a name and a system name
 * Gateway = APIcast self-managed
 * Authentication = OAuth 2.0
 * Click Create Service

 Once the service is created now configure the Integration. Click the `add the base URL of your API and save the configuration.` button. 
![Create](https://d26dzxoao6i3hh.cloudfront.net/items/3n2e340E3v1y080X2F11/%5B6dc51de91f1ee9c55e616d7576d3581e%5D_Screen%2520Shot%25202017-07-21%2520at%252011.24.36.png)

On this screen, you will need to configure both `Private Base URL` and `Staging Public Base URL`

* Private Base URL = `https://echo-api.3scale.net:443` or your own backend system
* Staging Public Base URL = Your APIcast host. Example: `http://localhost:8080`

## Creating an Application in 3scale

For this example, we are going to manually create an applicate in the admin portal. `Developers > Accounts > Select your developer account > Applications > Create application`  
![Create](https://d26dzxoao6i3hh.cloudfront.net/items/2X242S2c1Q0g2C1z0v26/%5Bb27fea9f5c7c73a5d9d8780b0e3b5f61%5D_Screen%2520Shot%25202017-07-21%2520at%252011.00.39.png)

* Select the correct plan for your API
* Name your application
![save](https://d26dzxoao6i3hh.cloudfront.net/items/173M1f2L3b3I2k1x3m0H/%5B1217000698dbf21b7ba9f041e4bfbb97%5D_Screen%2520Shot%25202017-07-21%2520at%252011.04.29.png)

* Now you will have an application with a `Client ID` and `Client Secret`. Take note of these as we will need to sync these later.
![details](https://d26dzxoao6i3hh.cloudfront.net/items/1u0V3B3K0A0A3I473j1q/%5Bae9760cac9f6603814ea481059f9db46%5D_Screen%2520Shot%25202017-07-21%2520at%252011.08.47.png)

## Creating client in RHSSO
To sync the credentials between RHSSO and 3scale we need to create a new client in RHSSO. To start we need to create an initial access token so we can register a new client. 

You can do this by selecting your realm and following `Realm Settings > Client Registration > Initial Access Tokens > Create`

![details](https://d26dzxoao6i3hh.cloudfront.net/items/2I1R3D1r3l3h1a380L3u/%5B76d2617626aec1b74574ac974f385206%5D_Screen%2520Shot%25202017-07-21%2520at%252013.02.26.png)

Once you have the required token you need to run the below cURL command adding the required values:

* Client ID (from 3scale)
* Client Secret (from 3scale)
* Access Token (from above)
* RHSSO host & port
* RHSSO realm (Default is `master`)

```
curl -X POST \
    -d '{ "clientId": "{CLIENT ID}", "secret":"{CLIENT SECRET}" }' \
    -H "Content-Type:application/json" \
    -H "Authorization: bearer {ACCESS TOKEN}" \
    http://{RHSSO HOST}:{RHSSO HOST}/auth/realms/{REALM}/clients-registrations/default
```
Once the client is created you will need to further configure it to allow client credentials. To do this in RHSSO select Clients > {new client} and configure the below settings:

* Enabled = ON
* Access Type = confidential
* Standard Flow Enabled = ON
* Service Accounts Enabled = ON
* Valid Redirect URIs = `/auth/realms/master/{CLIENT ID}/*`
* Base URL = `/auth/realms/master/{CLIENT ID}`

See an example below:
![details](https://d26dzxoao6i3hh.cloudfront.net/items/3v341s1k0e1S210d152c/%5Bef3e00bc119dc450da97b4c3fb04a681%5D_Screen%2520Shot%25202017-07-21%2520at%252013.12.53.png)

## Flow testing

### Sample Call For Token
Request to APIcast providing client ID and client secret

**Values to replace:**

* {APIcast HOST}
* {APIcast PORT}
* {CLIENT ID}
* {CLIENT SECRET}

```
curl -i -H 'Content-Type: application/x-www-form-urlencoded' -X POST 'http://{APIcast HOST}:{APIcast PORT}/oauth/token' -d 'grant_type=client_credentials&client_id={CLIENT ID}&client_secret={CLIENT SECRET}'
```

**Example Response:**

``` 
HTTP/1.1 200 OK
Content-Type: application/json
Connection: keep-alive
Content-Length: 2673
Set-Cookie: KC_RESTART=; Version=1; Expires=Thu, 01-Jan-1970 00:00:10 GMT; Max-Age=0; Path=/auth/realms/master; HttpOnly
Connection: keep-alive
Date: Fri, 21 Jul 2017 03:20:09 GMT
Server: WildFly/10
X-Powered-By: Undertow/1
```

```
{"access_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJjM3k1d0pJdzhfZzhXQnU1TXdIYkFKanVhMkRfLUpib1RrWGVadC03SzkwIn0.eyJqdGkiOiJhM2RkZjFiZi03YjczLTQ2MWYtYTE5Yi1kY2M5ZDFmMTg5NzMiLCJleHAiOjE1MDkyNDcyMDksIm5iZiI6MCwiaWF0IjoxNTAwNjA3MjA5LCJpc3MiOiJodHRwOi8vZWMyLTU0LTY2LTIxMi0xNC5hcC1zb3V0aGVhc3QtMi5jb21wdXRlLmFtYXpvbmF3cy5jb206MzAwMC9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJiMTdjMzNhMSIsInN1YiI6IjdlZWE1NDMxLTFmN2MtNDAzMi04NWUwLWNlYzIyZjNjYThlMyIsInR5cCI6IkJlYXJlciIsImF6cCI6ImIxN2MzM2ExIiwiYXV0aF90aW1lIjowLCJzZXNzaW9uX3N0YXRlIjoiYjAxMTExN2UtZGZlOS00ZjdlLWE4NmYtMGY4ZmNkMjNhODNkIiwiYWNyIjoiMSIsImFsbG93ZWQtb3JpZ2lucyI6W10sInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sImNsaWVudEhvc3QiOiIxNzIuMTcuMC4xIiwiY2xpZW50SWQiOiJiMTdjMzNhMSIsInByZWZlcnJlZF91c2VybmFtZSI6InNlcnZpY2UtYWNjb3VudC1iMTdjMzNhMSIsImNsaWVudEFkZHJlc3MiOiIxNzIuMTcuMC4xIiwiZW1haWwiOiJzZXJ2aWNlLWFjY291bnQtYjE3YzMzYTFAcGxhY2Vob2xkZXIub3JnIn0.ODSvk95TKPNZ1zWdDOcwlnWN8S4t9vSek0yzHrK6JnQ6gcUGZXpsFz8B6qAo1upuLh1hUgYO81d4SBUftxgnCuay24DLg0HRWuD1rk20BYGhtTKwD9kYtWrosedwwM6U5YeBrgurphNeQxUoQghmDQcQhzkNNCJRevA8n7sgASdbXy2P4s0I3oX0PiVcepqDV2Z3QA3GJ1r9tJO4xZQhGvUo_LSuas5GA9P0R44huftT0qQ-TB4ONG3zvfNInfosYrZDYMROFLeVGshpaqr84Dz2XgY9q0RdjBxmzBEKsDil3vDlqmkQPDU4jJ_YimFQyVfdvdVgpfRsLAzUZrnqTA","expires_in":8640000,"refresh_expires_in":1800,"refresh_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJjM3k1d0pJdzhfZzhXQnU1TXdIYkFKanVhMkRfLUpib1RrWGVadC03SzkwIn0.eyJqdGkiOiI0ZmRlODIzMC04MmM5LTRhYWUtOTc3Ny1jZjBhMmNkNmIyNTQiLCJleHAiOjE1MDA2MDkwMDksIm5iZiI6MCwiaWF0IjoxNTAwNjA3MjA5LCJpc3MiOiJodHRwOi8vZWMyLTU0LTY2LTIxMi0xNC5hcC1zb3V0aGVhc3QtMi5jb21wdXRlLmFtYXpvbmF3cy5jb206MzAwMC9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJiMTdjMzNhMSIsInN1YiI6IjdlZWE1NDMxLTFmN2MtNDAzMi04NWUwLWNlYzIyZjNjYThlMyIsInR5cCI6IlJlZnJlc2giLCJhenAiOiJiMTdjMzNhMSIsImF1dGhfdGltZSI6MCwic2Vzc2lvbl9zdGF0ZSI6ImIwMTExMTdlLWRmZTktNGY3ZS1hODZmLTBmOGZjZDIzYTgzZCIsInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX19.amG9IR_03gEmqetxVgLXcJMGzwvQSos5ytl6n6f4cUtaEK4Uyi3ajqbafaWafI8Gb25pL93pjZLdhMnQ-pJy9IgP6R6PCDaW5L4UcBouX2hZO30JnkN0JXwBuzsuBaE4rqT7DPjr4mHD8B_hzfq08UtKC9JgfRQN-ODMEQUBjemBPYnAqsAFWAc5DoYJqoeEGeR7VzyiRjZsWoBY0nxwdciLfJbYadc2G8MDidY-4QXuy9xSLc0034Lyq9vMHmAoW7eeL2E-pEBEhUHaYTpr_HEeI_YhSn7pKcQsAliND-TPC2tBI-enMNtI_ddN54fuj6gp28zxrqim84hM7h3eeA","token_type":"bearer","not-before-policy":0,"session_state":"b011117e-dfe9-4f7e-a86f-0f8fcd23a83d"}
```

### Sample Call using Token

Using the token it is now possible to send a request to the protected resource.

```
curl -X GET \
    -H 'accept-charset: UTF-8' \
    -H 'authorization: Bearer {ACCESS TOKEN}' \
    -H 'cache-control: no-cache' \
    http://{APIcast HOST}:{APIcast PORT} /{PATH}
```


