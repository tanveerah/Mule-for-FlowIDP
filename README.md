# MuleFlow IDP — Flow IDP as an Invocable Service

A MuleSoft 4 application that exposes Salesforce **Flow IDP** document extraction as a set of REST endpoints. It handles OAuth2 authentication via the client credentials grant, caches the access token in the Object Store, and invokes the `extractDataFromDocument` action on demand.

---

## Table of Contents

- [Architecture](#architecture)
- [Endpoints](#endpoints)
- [Flow Details](#flow-details)
- [Configuration](#configuration)
- [Error Handling](#error-handling)
- [Prerequisites](#prerequisites)

---

## Architecture

```
Client
  │
  ├─ POST /token         → Salesforce OAuth2 → access_token cached in Object Store
  ├─ GET  /token/cached  → Object Store read → return cached token
  └─ POST /extract-data  → Object Store read → Salesforce Flow IDP → return extracted data
```

The token and extraction flows are intentionally decoupled. Obtain a token once via `POST /token`, then call `POST /extract-data` as many times as needed without re-authenticating until the token expires.

---

## Endpoints

| Method | Path | Flow | Description |
|--------|------|------|-------------|
| `POST` | `/token` | `get-salesforce-access-token-flow` | Obtain a Salesforce OAuth2 token and cache it |
| `GET` | `/token/cached` | `get-cached-salesforce-token-flow` | Read the cached token from the Object Store |
| `POST` | `/extract-data` | `extract-data-flow` | Submit a document to Flow IDP using the cached token |

All endpoints listen on port `8081`.

---

## Flow Details

### `get-salesforce-access-token-flow` — `POST /token`

```
POST /token
  → Build URL-encoded form body
  → POST to Salesforce /services/oauth2/token
  → Extract access_token, token_type, instance_url
  → Store access_token in Object Store
  → Return token response
```

| Step | Description |
|------|-------------|
| **HTTP Listener** | Accepts `POST /token` with a JSON body containing `clientId` and `clientSecret`. |
| **Transform** | Combines `grant_type` (from `config.yaml`) with `clientId` and `clientSecret` into a `application/x-www-form-urlencoded` payload. |
| **HTTP Request** | POSTs credentials to `https://<your-salesforce-url>/services/oauth2/token`. |
| **Transform** | Extracts `access_token`, `token_type`, and `instance_url` from the Salesforce response. |
| **Object Store** | Persists `access_token` under key `salesforce_access_token` in the default in-memory Object Store. |

---

### `get-cached-salesforce-token-flow` — `GET /token/cached`

```
GET /token/cached
  → Read salesforce_access_token from Object Store
  → Return token (or error if not found)
```

| Step | Description |
|------|-------------|
| **HTTP Listener** | Accepts `GET /token/cached`. No request body required. |
| **Object Store** | Reads `salesforce_access_token` into `vars.accessToken`. Sets variable to `null` on `OS:KEY_NOT_FOUND`. |
| **Choice Router** | Returns `{"access_token": "..."}` if found; returns a `Token Not Found` error if not. |

---

### `extract-data-flow` — `POST /extract-data`

```
POST /extract-data  (multipart/form-data, file part)
  → Base64-encode uploaded file
  → Read access_token from Object Store
  → POST to Salesforce Flow IDP extractDataFromDocument
  → Return IDP extraction response
```

| Step | Description |
|------|-------------|
| **HTTP Listener** | Accepts `POST /extract-data` with a `multipart/form-data` body. File must be in a part named `file`. |
| **Transform** | Reads the `file` part, extracts MIME type and filename, and Base64-encodes the binary content. |
| **Set Variables** | Saves `fileBase64Content` and `fileMimeType` to flow variables before the Object Store retrieve replaces the payload. |
| **Object Store** | Reads `salesforce_access_token` into `vars.accessToken`. Returns a `401` error if no token is cached. |
| **Transform** | Builds the JSON request body with `fileContent`, `mimeType`, and `documentProcessingConfiguration` (from `config.yaml`). |
| **HTTP Request** | POSTs to `/services/data/<apiVersion>/actions/standard/extractDataFromDocument` with `Authorization: Bearer <token>`. |
| **Transform** | Passes through the full IDP response including `extractedData`, confidence scores, and `needsManualReview`. |

---

## Configuration

Edit `src/main/resources/config.yaml` before running the application. Copy `config.yaml.local` (gitignored) to store your real values locally.

| Property | Default | Description |
|----------|---------|-------------|
| `salesforce.host` | `<your-salesforce-url>` | Your Salesforce instance hostname |
| `salesforce.port` | `443` | HTTPS port |
| `salesforce.basePath` | `/services/oauth2` | OAuth2 base path |
| `salesforce.grantType` | `client_credentials` | OAuth2 grant type |
| `salesforce.apiVersion` | `v67.0` | Salesforce API version |
| `salesforce.idpConfigName` | `<your-idp-config-name>` | Name of your Flow IDP document extraction configuration |

---

## API Reference

### `POST /token`

Obtains a Salesforce OAuth2 access token using the client credentials grant and stores it in the Object Store.

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

**Response — 200 OK**
```json
{
  "access_token": "00D...",
  "token_type": "Bearer",
  "instance_url": "https://<your-salesforce-url>"
}
```

---

### `GET /token/cached`

Returns the access token currently held in the Object Store without re-authenticating.

**Request**
```http
GET http://localhost:8081/token/cached
```

**curl**
```bash
curl -X GET http://localhost:8081/token/cached
```

**Response — 200 OK (token found)**
```json
{
  "access_token": "00D..."
}
```

**Response — 200 OK (no token cached)**
```json
{
  "error": "Token Not Found",
  "message": "No Salesforce access token found in Object Store. Call POST /token first."
}
```

---

### `POST /extract-data`

Submits a document to Salesforce Flow IDP for data extraction. Reads the cached access token from the Object Store — call `POST /token` first.

Supported file types: **PDF**, **PNG**, **JPEG**.

**Request**
```http
POST http://localhost:8081/extract-data
Content-Type: multipart/form-data

file=<binary file>
```

**curl**
```bash
curl -X POST http://localhost:8081/extract-data \
  -F "file=@/path/to/your/document.pdf"
```

**Response — 200 OK**
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

## Error Handling

All error responses follow the same structure:

```json
{
  "error": "<error type>",
  "message": "<human-readable description>",
  "details": "<original error description>"
}
```

| Endpoint | Error | Likely Cause |
|----------|-------|--------------|
| `POST /token` | `HTTP:UNAUTHORIZED` | Invalid `clientId` or `clientSecret` |
| `POST /token` | `HTTP:CONNECTIVITY` | Cannot reach the Salesforce OAuth2 endpoint |
| `GET /token/cached` | `ANY` | Unexpected error during Object Store retrieval |
| `POST /extract-data` | `OS:KEY_NOT_FOUND` | No token cached — call `POST /token` first |
| `POST /extract-data` | `HTTP:UNAUTHORIZED` | Cached token is expired — call `POST /token` to refresh |
| `POST /extract-data` | `HTTP:BAD_REQUEST` | Unsupported file type or incorrect IDP configuration name |
| `POST /extract-data` | `MULE:EXPRESSION` | Missing or malformed `file` part in the multipart body |
| `POST /extract-data` | `ANY` | Unexpected error during file processing or IDP call |

---

## Prerequisites

- MuleSoft Anypoint Studio 7+ or Mule Runtime 4.x
- A Salesforce Connected App with the **Client Credentials** OAuth flow enabled
- A document extraction configuration created in Salesforce Flow IDP; set its name as the value of `salesforce.idpConfigName` in `config.yaml`
- `mule-objectstore-connector` 1.3.0 (declared in `pom.xml`)
