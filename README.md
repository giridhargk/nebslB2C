# NEBSL REST API Integration Guide

> Source documentation: NEBSL Cloud REST API docs  

## Overview

This repository contains a Git-friendly Markdown version of the NEBSL REST API documentation, organized for developers who want a quick implementation reference.

Based on the accessible documentation summary, the API supports:

- JSON-based requests and responses
- Data compression
- Authentication and session-based access
- Order operations such as:
  - Place Order
  - Modify Order
  - Cancel Order
- Reports such as:
  - Order Book
  - Trade Book
  - Position
  - Holding
  - Limit
- Portfolio information management
- Scrip Master details

---

## Basic Integration Flow

The documentation indicates the following high-level onboarding flow for implementation:

1. **Login**
   - Make the login API call using your authorized credentials.
   - Store the returned session/authentication details securely.

2. **Maintain Session**
   - Reuse the session or token returned by the login API for subsequent requests.
   - Ensure session validity before making trade or report requests.

3. **Call Functional APIs**
   - After successful login, call the required APIs depending on your use case:
     - Order placement
     - Order modification
     - Order cancellation
     - Order book and trade book retrieval
     - Portfolio and holdings retrieval
     - Limits and positions
     - Scrip master and related reference data

4. **Handle Standard Response Structure**
   - Parse the API response uniformly.
   - Centralize success/error handling in one place in your application.

---

## Request Conventions

From the available documentation summary:

- **GET** and **DELETE** request parameters are passed as **query parameters**
- **POST** and **PUT** request parameters are passed as **JSON body**
- Content type is expected to be **`application/json`**

### Example patterns

#### GET
```http
GET /api/example?clientCode=ABC123&exchange=NSE
```

#### POST
```http
POST /api/example
Content-Type: application/json

{
  "clientCode": "ABC123",
  "exchange": "NSE"
}
```

---

## Response Structure

The published documentation includes a dedicated **Response Structure** section.  
A practical reusable structure for implementation is shown below:

```json
{
  "status": true,
  "message": "Request processed successfully",
  "data": {}
}
```

### Recommended handling

- `status` → whether the request succeeded
- `message` → human-readable success or error message
- `data` → actual payload returned by the API

> Exact field names and nested objects may vary by endpoint. Validate each endpoint response during implementation.

---

## Functional Areas

## 1. Authentication

Used to establish a valid session before accessing trading or reporting APIs.

Typical responsibilities:

- User login
- Session generation
- Session validation
- Logout or session termination

### Implementation notes

- Keep secrets out of source control
- Store session tokens securely
- Add automatic re-login or refresh handling where needed
- Centralize auth logic in a dedicated client/service layer

---

## 2. Order Management

The documentation summary explicitly mentions:

- **Place Order**
- **Modify Order**
- **Cancel Order**

### Typical order lifecycle

1. Validate session
2. Build order payload
3. Submit order
4. Parse acknowledgement / order reference
5. Track order status using Order Book or Trade Book APIs
6. Handle rejects, partial fills, or exchange-side failures

### Suggested internal module structure

```text
src/
  api/
    auth_client.*
    order_client.*
    report_client.*
  models/
    auth.*
    order.*
    report.*
  services/
    session_service.*
    order_service.*
```

---

## 3. Reports

The documentation summary lists the following report categories:

- **Order Book**
- **Trade Book**
- **Position**
- **Holding**
- **Limit**

### Common usage

- Fetch order history
- Track completed trades
- Retrieve live positions
- Show demat/portfolio holdings
- Check client trading limits and margins

### Recommended implementation pattern

- Wrap each report in a dedicated function/service
- Normalize response mapping into internal DTOs/models
- Add retry logic for transient failures
- Log raw responses safely for debugging

---

## 4. Portfolio Information Management

The documentation summary mentions **Portfolio information management**.

This usually includes:

- Portfolio-level summary retrieval
- Client asset visibility
- Security-wise holdings
- Position and valuation access
- Investment overview components

---

## 5. Scrip Master Details

The documentation summary mentions **Scrip Master Details**.

This is usually used to:

- Fetch instrument metadata
- Resolve symbol/token/security identifiers
- Validate tradable contracts or scrips
- Map internal instrument references

---

## Suggested README Sections for Production Repositories

If you are adding this into a Git repository, this structure works well:

```md
# Project Name

## Environment Variables
## Authentication
## API Endpoints
## Request/Response Examples
## Error Handling
## Retry Strategy
## Logging
## Security Notes
## Testing
```

---

## Example API Client Wrapper

Below is a generic example in JavaScript/TypeScript style:

```ts
type ApiResponse<T> = {
  status: boolean;
  message: string;
  data: T;
};

async function apiRequest<T>(
  url: string,
  method: "GET" | "POST" | "PUT" | "DELETE",
  body?: unknown
): Promise<ApiResponse<T>> {
  const options: RequestInit = {
    method,
    headers: {
      "Content-Type": "application/json"
    }
  };

  if (body && (method === "POST" || method === "PUT")) {
    options.body = JSON.stringify(body);
  }

  const response = await fetch(url, options);
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  return response.json() as Promise<ApiResponse<T>>;
}
```

---

## Error Handling Recommendations

Implement these checks consistently:

- HTTP status validation
- API-level `status` validation
- Empty `data` handling
- Unauthorized session handling
- Timeout and retry handling
- Structured logs for trading/reporting calls

### Suggested categories

- Authentication errors
- Validation errors
- Business rule failures
- Exchange/order rejections
- Network timeouts
- Server-side internal errors

---

## Security Recommendations

- Never commit credentials to Git
- Use environment variables or a secret manager
- Mask client identifiers in logs
- Encrypt stored session artifacts where required
- Add rate limiting and retry backoff
- Keep audit logs for critical trade actions

---

## Testing Checklist

- [ ] Login success flow
- [ ] Login failure flow
- [ ] Session expiry flow
- [ ] Place order success
- [ ] Place order rejection
- [ ] Modify order success/failure
- [ ] Cancel order success/failure
- [ ] Order Book fetch
- [ ] Trade Book fetch
- [ ] Position fetch
- [ ] Holding fetch
- [ ] Limit fetch
- [ ] Invalid request body handling
- [ ] Network timeout handling

---

## Repository Usage Example

```bash
git clone <your-repo-url>
cd <your-repo>
cp .env.example .env
```

---

## Important Note

This README is a **developer-friendly Markdown conversion based on the publicly visible documentation summary available from the NEBSL REST API page**.

During this conversion, the original documentation site was not fully machine-readable from my side, so this file is a **clean implementation README draft**, not a verbatim endpoint-by-endpoint export.

For a complete production reference, you should still verify:

- Exact endpoint paths
- Required headers
- Authentication field names
- Mandatory request fields
- Exact response schema for each endpoint
- Error codes and exchange-specific behaviors

---

## Recommended Next Step

Use this file as the root `README.md`, then extend it with:

- Exact endpoint list
- Full request/response payload samples
- Sandbox/production base URLs
- Authentication headers
- Error code reference
- SDK examples for your preferred language

