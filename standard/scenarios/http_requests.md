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
| `duration.ms` | number | Duration of the request in milliseconds | 150 |

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
  "duration.ms": 45,
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
| `http.response.status_code` | number | HTTP response status code received | 200 |
| `peer.service` | string | Name of the service being called | "account-lookup-service" |
| `duration.ms` | number | Duration of the call in milliseconds | 230 |

### Example
```json
{
  "level": "INFO",
  "message": "Outgoing request to Account Lookup",
  "http.request.method": "GET",
  "url.full": "http://als/participants/123",
  "http.response.status_code": 200,
  "peer.service": "account-lookup",
  "duration.ms": 120
}
```
