# HTTP Request Logging Standard

## Overview
This standard defines the required fields and practices for logging HTTP requests (both incoming and outgoing). Adhering to this standard ensures consistent observability of API traffic and service interactions.

## Incoming Requests (Server Side)

All incoming HTTP requests must be logged at the completion of the request.

### Required Fields

| Field Name | Type | Description | Example |
|------------|------|-------------|---------|
| `http.request.method` | string | HTTP request method | "POST", "GET" |
| `url.full` | string | Full request URL | "https://api.mojaloop.io/transfers" |
| `url.path` | string | The target path | "/transfers" |
| `server.address` | string | The server address (Host) | "api.mojaloop.io" |
| `http.response.status_code` | number | HTTP response status code | 200, 400, 500 |
| `http.route` | string | The matched route path (low cardinality) | "/transfers/:id" |
| `user_agent.original` | string | User agent string | "Mozilla/5.0..." |
| `client.address` | string | IP address of the client | "192.168.1.1" |
| `request.id` | string | Unique request identifier (Trace ID) | "req-123xyz" |
| `http.server.request.duration` | number | Duration of the request in seconds | 0.150 |
| `error.type` | string | Error type if request failed (conditional) | "timeout", "connection_error" |

> **Note:** We use `http.server.request.duration` (in seconds) instead of a custom `duration.ms` attribute because OTel semantic conventions define duration as the measured value of a histogram metric, not an attribute. Using the standard metric name aligns with OTel tooling and dashboards.

### Recommended Fields
* `http.request_content_length`
* `http.response_content_length`

### FSPIOP Headers
For Mojaloop-specific API calls, the following FSPIOP headers must be logged as attributes if present.

| Header | Attribute Key | Description | Handling |
|--------|---------------|-------------|----------|
| `FSPIOP-Source` | `fspiop.source` | Sender FSP ID | Log value. |
| `FSPIOP-Destination` | `fspiop.destination` | Recipient FSP ID | Log value. |
| `FSPIOP-Signature` | `fspiop.signature` | Request Integrity Signature | **Hash** the value (do not log full JWS). |
| `FSPIOP-URI` | `fspiop.uri` | Service URI used for signature | Log value. |
| `FSPIOP-HTTP-Method` | `fspiop.method` | Service Method used for signature | Log value. |
| `FSPIOP-Encryption` | `fspiop.encryption` | Encryption header | Log metadata/algorithm only. |

### Example

```json
{
  "level": "INFO",
  "message": "Incoming request specific_route",
  "http.request.method": "POST",
  "http.route": "/transfers",
  "http.response.status_code": 201,
  "http.server.request.duration": 0.045,
  ...
}
```

## Outgoing Requests (Client Side)

All outgoing HTTP requests made by the service must be logged.

### Required Fields

| Field Name | Type | Description | Example |
|------------|------|-------------|---------|
| `http.request.method` | string | HTTP request method | "GET" |
| `url.full` | string | Full request URL | "https://external-service.com/api" |
| `http.response.status_code` | number | HTTP response status code received (conditional) | 200 |
| `error.type` | string | Error type if request failed (conditional) | "timeout", "ECONNREFUSED" |
| `service.peer.name` | string | Logical name of the remote service being called | "account-lookup-service" |
| `http.client.request.duration` | number | Duration of the call in seconds | 0.230 |

> **Note:** `http.response.status_code` is included only when a response is received. `error.type` is included only when the request fails (e.g., timeout, connection refused).

> **Note on `service.peer.name`:** This attribute replaces the deprecated `peer.service` attribute per [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/general/attributes/).

### Example (Success)
```json
{
  "level": "INFO",
  "message": "Outgoing request to Account Lookup",
  "http.request.method": "GET",
  "url.full": "http://als/participants/123",
  "http.response.status_code": 200,
  "service.peer.name": "account-lookup",
  "http.client.request.duration": 0.120
}
```

### Example (Error)
```json
{
  "level": "ERROR",
  "message": "Outgoing request to Account Lookup failed",
  "http.request.method": "GET",
  "url.full": "http://als/participants/123",
  "error.type": "ECONNREFUSED",
  "service.peer.name": "account-lookup",
  "http.client.request.duration": 5.001
}
```
