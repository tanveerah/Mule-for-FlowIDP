# MuleFlow IDP — Salesforce OAuth2 + Flow IDP Service

A MuleSoft 4 application that authenticates with Salesforce using the **client credentials** grant type, caches the access token in the Object Store, and submits documents to the **Salesforce Flow IDP** `extractDataFromDocument` action.

---

## Flow Overview

| Flow | Endpoint | Purpose |
|------|----------|---------|
| `get-salesforce-access-token-flow` | `POST /token` | Obtain a Salesforce OAuth2 token and cache it |
| `get-cached-salesforce-token-flow` | `GET /token/cached` | Read the cached token from the Object Store |
| `extract-data-flow` | `POST /extract-data` | Submit a file to Flow IDP using the cached token |

---

### `get-salesforce-access-token-flow`

```
POST /token  →  Build form body  →  POST Salesforce /services/oauth2/token  →  Store in Object Store  →  Return token
```

| Step | Description |
|------|-------------|
| **HTTP Listener** | Listens on `0.0.0.0:8081`. Accepts `POST /token` with a JSON body containing `clientId` and `clientSecret`. |
| **Transform — Build OAuth2 Request Body** | Reads `grant_type` from `config.yaml` and combines it with `clientId`/`clientSecret` into a `application/x-www-form-urlencoded` payload. |
| **HTTP Request — Salesforce Token Endpoint** | POSTs the form body to `https://<your-salesforce-url>/services/oauth2/token`. |
| **Transform — Extract Token** | Parses the Salesforce response and returns `access_token`, `token_type`, and `instance_url` as JSON. |
| **Object Store — Store Token** | Stores `access_token` under key `salesforce_access_token` in the default in-memory Object Store. |

---

### `get-cached-salesforce-token-flow`

```
GET /token/cached  →  Retrieve from Object Store  →  Return token (or 404 if not found)
```

| Step | Description |
|------|-------------|
| **HTTP Listener** | Accepts `GET /token/cached`. |
| **Object Store — Retrieve Token** | Reads `salesforce_access_token` from the Object Store into `vars.accessToken`. Handles `OS:KEY_NOT_FOUND` by setting the variable to `null`. |
| **Choice — Token Exists?** | Returns `{"access_token": "..."}` if found; returns a `Token Not Found` error body if not. |

---

### `extract-data-flow`

```
POST /extract-data  →  Base64-encode file  →  Retrieve token from Object Store  →  POST to Flow IDP  →  Return IDP response
```

| Step | Description |
|------|-------------|
| **HTTP Listener** | Accepts `POST /extract-data` with a `multipart/form-data` body. The file must be in a part named `file`. |
| **Transform — Extract and Base64-encode File** | Reads the `file` part, extracts MIME type and filename, and Base64-encodes the binary content. |
| **Set Variables** | Saves `fileBase64Content` and `fileMimeType` to flow variables before the Object Store retrieve overwrites the payload. |
| **Object Store — Retrieve Token** | Reads `salesforce_access_token` into `vars.accessToken`. Propagates a `401` error if no token is cached. |
| **Transform — Build IDP Request Body** | Constructs the JSON body with `fileContent`, `mimeType`, and `documentProcessingConfiguration` (from `config.yaml`). |
| **HTTP Request — Salesforce Flow IDP** | POSTs to `/services/data/<apiVersion>/actions/standard/extractDataFromDocument` with `Authorization: Bearer <token>`. |
| **Transform — Parse IDP Response** | Returns the full IDP response array including `extractedData`, confidence scores, and `needsManualReview`. |

---

## Request & Response

### `POST /token`

**Request**
```http
POST http://localhost:8081/token
Content-Type: application/json

{
  "clientId": "<your-connected-app-client-id>",
  "clientSecret": "<your-connected-app-client-secret>"
}
```

**curl**
```bash
curl -X POST http://localhost:8081/token \
  -H "Content-Type: application/json" \
  -d '{
    "clientId": "<your-connected-app-client-id>",
    "clientSecret": "<your-connected-app-client-secret>"
  }'
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

### `GET /token/cached`

**Request**
```http
GET http://localhost:8081/token/cached
```

**curl**
```bash
curl -X GET http://localhost:8081/token/cached
```

**Response (token found)**
```json
{
  "access_token": "00D..."
}
```

**Response (no token cached)**
```json
{
  "error": "Token Not Found",
  "message": "No Salesforce access token found in Object Store. Call POST /token first."
}
```

---

### `POST /extract-data`

**Request**
```http
POST http://localhost:8081/extract-data
Content-Type: multipart/form-data

file=<binary file content>   (PDF, PNG, or JPEG)
```

**curl**
```bash
curl -X POST http://localhost:8081/extract-data \
  -F "file=@/path/to/your/document.pdf"
```

**Response**
```json
[
  {
    "actionName": "extractDataFromDocument",
    "isSuccess": true,
    "outputValues": {
      "extractedData": { ... },
      "extractedDataJson": "...",
      "needsManualReview": false
    }
  }
]
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
| `salesforce.apiVersion` | `v67.0` |
| `salesforce.idpConfigName` | `Extract_from_DL` |

---

## Error Handling

| Flow | Error | Cause |
|------|-------|-------|
| `get-salesforce-access-token-flow` | `HTTP:UNAUTHORIZED` | Invalid `clientId` or `clientSecret` |
| `get-salesforce-access-token-flow` | `HTTP:CONNECTIVITY` | Cannot reach the Salesforce OAuth2 endpoint |
| `get-cached-salesforce-token-flow` | `ANY` | Unexpected error during Object Store retrieval |
| `extract-data-flow` | `OS:KEY_NOT_FOUND` | No token cached — call `POST /token` first |
| `extract-data-flow` | `HTTP:UNAUTHORIZED` | Cached token is expired — call `POST /token` to refresh |
| `extract-data-flow` | `HTTP:BAD_REQUEST` | Unsupported file type or incorrect IDP config name |
| `extract-data-flow` | `MULE:EXPRESSION` | Missing or malformed `file` part in the multipart body |
| `extract-data-flow` | `ANY` | Unexpected error during file processing or IDP call |

All errors return a structured JSON body with `error`, `message`, and `details` fields.

---

## Prerequisites

- MuleSoft Anypoint Studio 7+ or Mule Runtime 4.x
- A Salesforce Connected App with the **Client Credentials** OAuth flow enabled
- A Salesforce Flow IDP configuration named `Extract_from_DL` (or update `salesforce.idpConfigName` in `config.yaml`)
- `mule-objectstore-connector` 1.3.0 (declared in `pom.xml`)
