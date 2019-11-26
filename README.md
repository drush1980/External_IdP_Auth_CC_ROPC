# External IdP Auth Integration Proxy

This is an Apigee API Proxy that shows how to integrate Apigee with a third party identity management platform (in this example [Okta](https://www.okta.com/)).  This proxy supports requesting OAuth tokens from the IdP using either [client credentials](https://tools.ietf.org/html/rfc6749#section-4.4) or [resource owner password credentials](https://tools.ietf.org/html/rfc6749#section-4.3) flows.


## Disclaimer

This example is not an official Google product, nor is it part of an official Google product.


## Notes

* The proxy exposes two different token endpoints that demonstrate how Apigee can be used as the issuing token authority (i.e. where access tokens are minted by Apigee), or how the external IdP generated token can be stored and subsequently validated by Apigee.
* This proxy was designed to work with the Okta OAuth 2 API. In order to use this proxy as it, you must have an Okta developer account, or sign up for one [here](https://developer.okta.com/signup/).
* For Okta application setup steps and required inputs for the client credentials flow see: https://developer.okta.com/docs/guides/implement-client-creds/overview/
* For Okta application setup steps and required inputs for the resource owner password credentials flow see: https://developer.okta.com/docs/guides/implement-password/overview/

## Setup

You will need an Apigee Edge login to use this proxy.  If you don't have one you may sign up for a trial account [here](https://login.apigee.com/sign__up).

Before exercising this proxy, you need to:

1. Create a [target server](https://docs.apigee.com/api-platform/deploy/load-balancing-across-backend-servers) entry in Apigee Edge named `okta`.  Set the value to your [Okta account domain](https://developer.okta.com/docs/guides/find-your-domain/overview/).

2. Deploy the proxy bundle to your Apigee Edge environment (see [Deploying the Proxy](#deploy) below).


> The following steps use the Apigee Management API to create products, developers, apps and keys.  To invoke you will need your Apigee [organization and environment](https://docs.apigee.com/api-platform/fundamentals/apigee-edge-organization-structure) names, and your [admin credentials](https://docs.apigee.com/api-platform/system-administration/basic-auth?hl=en).

Run the following calls, replacing the values with your Apigee org + environment names, and encoded admin credentials:

3. Create an [API Product](https://docs.apigee.com/api-platform/publish/what-api-product):

```
curl --request POST \
  --url https://api.enterprise.apigee.com/v1/organizations/{yourApigeeOrg}/apiproducts \
  --header 'authorization: Basic {b64 credentials}' \
  --header 'content-type: application/json' \
  --data '{
  "apiResources": [
  ],
  "approvalType": "auto",
  "attributes": [
    {
      "name": "access",
      "value": "public"
    }
  ],
  "description": "Example External Auth Product",
  "displayName": "Example External Auth Product",
  "environments": [
    "{yourApigeeEnvironment}"
  ],
  "name": "Example-External-Auth-Product",
  "proxies": [
    "External_IdP_Auth_CC_ROPC"
  ],
  "scopes": [
  ]
}'
```

4. Create a [developer](https://docs.apigee.com/api-platform/publish/adding-developers-your-api-product) account (you may replace with your own email and name if desired):

```
curl --request POST \
  --url https://api.enterprise.apigee.com/v1/organizations/{yourApigeeOrg}/developers \
  --header 'authorization: Bearer {b64 credentials}' \
  --header 'content-type: application/json' \
  --data '{
 "email" : "developeremail@somedomain.com",
 "firstName" : "Joe",
 "lastName" : "Developer",
 "userName" : "developeremail@somedomain.com",
 "attributes" : []
}'
```

5. Create a developer [app](https://docs.apigee.com/api-platform/publish/creating-apps-surface-your-api), using developer email from step 4:

```
curl --request POST \
  --url https://api.enterprise.apigee.com/v1/organizations/{yourApigeeOrg}/developers/developeremail@somedomain.com/apps \
  --header 'Authorization: Bearer {b64 credentials}' \
  --header 'content-type: application/json' \
  --data '{
  "name": "Example-External-Auth-App",
	"apiProducts": ["Example-External-Auth-Product"],
  "attributes": [
    {
      "name": "DisplayName",
      "value": "Example External Auth App"
    },
    {
      "name": "Notes",
      "value": ""
    }
  ],
  "callbackUrl": "",
  "scopes": [
  ]
}'
```

6. Add the external Okta application `client_id` and `client_secret` to the Apigee developer app:

```
curl --request POST \
  --url https://api.enterprise.apigee.com/v1/organizations/{yourApigeeOrg}/developers/developeremail@somedomain.com/apps/Example-External-Auth-App/keys/create \
  --header 'authorization: Bearer {b64 credentials}' \
  --header 'content-type: application/json' \
  --data '{
	"consumerKey": "{okta client_id}",
	"consumerSecret": "{okta client_secret}"
}'
```

7. Associate the API product to the imported key (note you must replace the Okta `client_id` in the path):

```
curl --request POST \
  --url https://api.enterprise.apigee.com/v1/organizations/{yourApigeeOrg}/developers/developeremail@somedomain.com/apps/Example-External-Auth-App/keys/{okta client_id} \
  --header 'authorization: Bearer {b64 credentials}' \
  --header 'content-type: application/json' \
  --data '{ "apiProducts": ["Example-External-Auth-Product"],
  "attributes": []
}'
```


## Testing the Proxy

The proxy exposes several endpoints.  You can test them using the following calls, and substituting the base64 encoded client_id and client_secret in an authorization header using [basic HTTP auth](https://tools.ietf.org/html/rfc7617).

### 1. `/token` - return a token minted by Apigee

The `RequestToken` flow proxies the token request to the IdP, and if successful Apigee will mint a _new_ opaque access token.  The access token returned from the external IdP will be attached as an attribute named `subject.idp.access_token`

Example client credentials request:

```
curl --request POST \
  --url https://{yourApigeeOrg}-{yourApigeeEnvironment}.apigee.net/v1/thirdPartyAuth/token \
  --header 'authorization: Basic {b64 client credentials}' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=client_credentials \
  --data scope=customScope
```

Example resource owner password credentials request (note you must pass valid Okta user credentials that are assigned to the application in Okta):

```
curl --request POST \
  --url https://{yourApigeeOrg}-{yourApigeeEnvironment}.apigee.net/v1/thirdPartyAuth/token \
  --header 'authorization: Basic {b64 client credentials}' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=password \
  --data scope=customScope \
  --data 'username=oktausername@somedomain.com' \
  --data password=oktaUserPassword
```

Example response:

```
{
  "refresh_token_expires_in": "0",
  "api_product_list": "[Example-External-Auth-Product]",
  "api_product_list_json": [
    "Example-External-Auth-Product"
  ],
  "organization_name": "exampleOrg",
  "developer.email": "developeremail@somedomain.com",
  "token_type": "BearerToken",
  "issued_at": "1574730664554",
  "client_id": "0oa1xxudrhQZ5rtJW357",
  "access_token": "VOMY4xSDTnMJaEWtXqjGlj0kbTk7",
  "application_name": "e159a84f-5435-4ad7-9a4c-b6578ead5d91",
  "scope": "",
  "subject.idp.access_token": "eyJraWQiOiJLblMwUzV1cUhSTWppODBPRWxCTVlydjJ4S1dHcmZQZHNsQzdTSXZ1OW9VIiwiYWxnIjoiUlMyNTYifQ.eyJ2ZXIiOjEsImp0aSI6IkFULlUyOGZnMW8tSWNkajBtb2F3RVFVMEl3aUY4aE01MmpXZWdkSTJ2OUs1TnMiLCJpc3MiOiJodHRwczovL2Rldi04NDgyNTUub2t0YS5jb20vb2F1dGgyL2RlZmF1bHQiLCJhdWQiOiJhcGk6Ly9kZWZhdWx0IiwiaWF0IjoxNTc0NzMwNjY0LCJleHAiOjE1NzQ3MzQyNjQsImNpZCI6IjBvYTF4eHVkcmhRWjVydEpXMzU3Iiwic2NwIjpbImN1c3RvbVNjb3BlIl0sInN1YiI6IjBvYTF4eHVkcmhRWjVydEpXMzU3In0.VN_5D1v8BWRd1up3ISerHaTLvkMhqhkyR8R9SK73q448LXBJbL_n8vCTmHe0XVraBg7YEppDOrgm7eg3k27yqkgP4yCjeghGIAqw4NZU8vcX8v9B704DOgAP1jl7OFQytMomZygI6W54MU_eMflqRuLvSLWwklWy-03rz7-yay1bmgqHr25jqs8RQRBxpfe_nE7eOPPYS7YSHzZBZ5ln69BJVrrlP-E7QOPDy795Y2H38Pv9jU80ppHRF9Q0jdeAkrhnorXiv8LKB7A0asVd-KEpIBhayrOxJsvabOAOfG6b1SWRJ82XF0gzuYn0T2l__z8VqwrfxYLTj1KhO_7I1A",
  "expires_in": "1799",
  "refresh_count": "0",
  "status": "approved"
}
```


### 2. `/exttoken` - store and return the _external_ IdP token

The `RequestTokenExternal` flow proxies the token request to the IdP, and if successful Apigee will store the access token obtained from the IdP itself.

Example client credentials request:

```
curl --request POST \
  --url https://{yourApigeeOrg}-{yourApigeeEnvironment}.apigee.net/v1/thirdPartyAuth/exttoken \
  --header 'authorization: Basic {b64 client credentials}' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=client_credentials \
  --data scope=customScope
```

Example resource owner password credentials request (note you must pass valid Okta user credentials that are assigned to the application in Okta):

```
curl --request POST \
  --url https://{yourApigeeOrg}-{yourApigeeEnvironment}.apigee.net/v1/thirdPartyAuth/exttoken \
  --header 'authorization: Basic {b64 client credentials}' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=password \
  --data scope=customScope \
  --data 'username=oktausername@somedomain.com' \
  --data password=oktaUserPassword
```

Example response:

```
{
  "refresh_token_expires_in": "0",
  "api_product_list": "[Example-External-Auth-Product]",
  "api_product_list_json": [
    "Example-External-Auth-Product"
  ],
  "organization_name": "exampleOrg",
  "developer.email": "developeremail@somedomain.com",
  "token_type": "BearerToken",
  "issued_at": "1574722863633",
  "client_id": "0oa1xxudrhQZ5rtJW357",
  "access_token": "eyJraWQiOiJLblMwUzV1cUhSTWppODBPRWxCTVlydjJ4S1dHcmZQZHNsQzdTSXZ1OW9VIiwiYWxnIjoiUlMyNTYifQ.eyJ2ZXIiOjEsImp0aSI6IkFULjdRSzB5aG4xNE93YXNrVGJ2UEx0a3BXTE5BSi01UVFGT2RKTHZlMGxwbEEiLCJpc3MiOiJodHRwczovL2Rldi04NDgyNTUub2t0YS5jb20vb2F1dGgyL2RlZmF1bHQiLCJhdWQiOiJhcGk6Ly9kZWZhdWx0IiwiaWF0IjoxNTc0NzIyODYzLCJleHAiOjE1NzQ3MjY0NjMsImNpZCI6IjBvYTF4eHVkcmhRWjVydEpXMzU3Iiwic2NwIjpbImN1c3RvbVNjb3BlIl0sInN1YiI6IjBvYTF4eHVkcmhRWjVydEpXMzU3In0.oQRWelCSu_pEQEUMuiML68aym6foqNA33ab8IxG6LkFOda-FfrxN9Y0rZBAGpKZN_8Xm6nk3f-1nccfkX0Zj9iy1UE-ZN735GXLlzJQBjqnDHkHFohMgPwk2Wkk-EzVISRxm-xDvuUfJlczRt5WfItwdk4sOeP-faRffkHquVv7ufZ4yiH1X58U_NFEhunpc08M_X5U-VbOowCCjINbLhuzjHT45VLokT3oHbRkkPSfVVidf4riV8Tawa2CmDEfui7BSJdIcKheNxDsCqcYPa9808h5JB2ijw_9ZV-TDyc9GgjIbynDbHFhUjA9QpTpGxEsshP5vqX-ZfRFw2wI-sQ",
  "application_name": "e159a84f-5435-4ad7-9a4c-b6578ead5d91",
  "scope": "",
  "expires_in": "1799",
  "refresh_count": "0",
  "status": "approved"
}
```


### 3. `/tokeninfo` - return information about a token

The `TokenInfo` flow uses the OAuth V2 policy to validate the token and then return the information contained within.

Example request:

```
curl --request GET \
  --url 'https://{yourApigeeOrg}-{yourApigeeEnvironment}.apigee.net/v1/thirdPartyAuth/tokeninfo?access_token={access_token}'
```

Example response for valid token:

```
{
  "access_token": "VOMY4xSDTnMJaEWtXqjGlj0kbTk7",
  "refresh_token": "",
  "developer_id": "exampleOrg@@@4e564e40-000c-41ed-a59f-734dcc219468",
  "developer_email": "developeremail@somedomain.com",
  "organization_name": "exampleOrg",
  "api_product_list": "[Example-External-Auth-Product]",
  "developer_app_name": "Example-External-Auth-Product",
  "expires_in": "558",
  "status": "approved",
  "refresh_token": "",
  "refresh_token_status": "",
  "refresh_token_expires_in": "0",
  "scope": "",
  "client_id": "0oa1xxudrhQZ5rtJW357",
  "subject.idp.issuer": "",
  "subject.idp.id": "",
  "subject.idp.method": "",
  "subject.name": ""
}
```


### 4. `/service1` - validate token and return a mock response

The `ProtectedService` flow uses the OAuth V2 policy to validate the token and then returns a mock response from a fake "service".

Example request:

```
curl --request GET \
  --url https://{yourApigeeOrg}-{yourApigeeEnvironment}.apigee.net/v1/thirdPartyAuth/service1 \
  --header 'authorization: Bearer {access_token}'
```

Example response for valid token:

```
{
  "status": "OK",
  "token": "valid",
  "response": "SUCCESS"
}
```


## <a name="deploying"></a>Deploying the Proxy

To deploy the proxy you may use the [apigeetool](https://www.npmjs.com/package/apigeetool) utility.  After installing, run the following command from the root of this project, replacing the values for username, password and the Apigee org/env with the appropriate values for your setup.

```
apigeetool deployproxy -u youremail@example.com -p yourPassword -o orgName -e test -n External_IdP_Auth_CC_ROPC -d .
```

## Support

This example is just an illustration. It is not a supported part of Apigee Edge.
If you have questions on it, or would like assistance with it, you can try
enquiring on [The Apigee Community Site](https://community.apigee.com).  There
is no service-level guarantee for responses to enquiries regarding this example.


## License

This material is copyright 2019 Google LLC.
and is licensed under the [Apache 2.0 License](LICENSE).

