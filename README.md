# MuleFlow IDP — Salesforce OAuth2 Token Service

A MuleSoft 4 application that acts as an Identity Provider (IDP) proxy, retrieving OAuth2 access tokens from Salesforce using the **client credentials** grant type.

---

## Flow Overview

### `get-salesforce-access-token-flow`

```
POST /token  →  Build form body  →  POST Salesforce /services/oauth2/token  →  Return token
```

| Step | Description |
|------|-------------|
| **HTTP Listener** | Listens on `0.0.0.0:8081`. Accepts `POST /token` with a JSON body containing `clientId` and `clientSecret`. |
| **Transform — Build OAuth2 Request Body** | Reads `grant_type` from `config.yaml` and combines it with `clientId`/`clientSecret` from the request body into a `application/x-www-form-urlencoded` payload. |
| **HTTP Request — Salesforce Token Endpoint** | POSTs the form body to `https://<your-salesforce-url>/services/oauth2/token` over HTTPS (port 443). |
| **Transform — Extract Token** | Parses the Salesforce response and returns `access_token`, `token_type`, and `instance_url` as JSON. |
| **Set Variable — Store Access Token** | Stores the retrieved `access_token` in a flow variable named `accessToken` for use by downstream processors. |

---

## Request & Response

**Request**
```http
POST http://localhost:8081/token
Content-Type: application/json

{
  "clientId": "<your-connected-app-client-id>",
  "clientSecret": "<your-connected-app-client-secret>"
}
```

**Response**
```json
{
  "access_token": "00D...",
  "token_type": "Bearer",
  "instance_url": "https://<your-salesforce-url>"
}
```

---

## Configuration

Settings are loaded from `src/main/resources/config.yaml`:

| Property | Value |
|----------|-------|
| `salesforce.host` | `<your-salesforce-url>` |
| `salesforce.port` | `443` |
| `salesforce.basePath` | `/services/oauth2` |
| `salesforce.grantType` | `client_credentials` |

---

## Error Handling

| Error | HTTP Status | Cause |
|-------|-------------|-------|
| `HTTP:UNAUTHORIZED` | 401 | Invalid `clientId` or `clientSecret` |
| `HTTP:CONNECTIVITY` | — | Cannot reach the Salesforce OAuth2 endpoint |

Both errors return a structured JSON body with `error`, `message`, and `details` fields.

---

## Prerequisites

- MuleSoft Anypoint Studio 7+ or Mule Runtime 4.x
- A Salesforce Connected App with the **Client Credentials** OAuth flow enabled
