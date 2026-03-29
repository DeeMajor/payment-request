# PayLink API Design

## Overview

PayLink is a mobile-first payment request platform that allows
individuals or small service providers to generate payment links and QR
codes.

Payments are processed via an external payment gateway (PayFast).

------------------------------------------------------------------------

# API Endpoints

## Create Payment Request

POST /payment-requests

Creates a payment request and returns a payment link and QR code.

### Request

``` json
{
  "amount": 150.00,
  "currency": "ZAR",
  "description": "Haircut",
  "expiresAt": "2026-03-20T18:00:00Z"
}
```

### Response

``` json
{
  "id": "req_abc123",
  "amount": 150.00,
  "currency": "ZAR",
  "status": "pending",
  "paymentLink": "https://paylink.app/pay/req_abc123",
  "qrCodeUrl": "https://paylink.app/qr/req_abc123",
  "createdAt": "2026-03-15T12:00:00Z"
}
```

------------------------------------------------------------------------

## Get Payment Request

GET /payment-requests/{id}

Returns details of a specific payment request.

### Response

``` json
{
  "id": "req_abc123",
  "amount": 150,
  "status": "pending",
  "paymentLink": "https://paylink.app/pay/req_abc123"
}
```

------------------------------------------------------------------------

## List Payment Requests

GET /payment-requests?page=1&pageSize=10

Returns a paginated list of payment requests.

### Response

``` json
{
  "data": [
    {
      "id": "req_abc123",
      "amount": 150,
      "status": "pending"
    },
    {
      "id": "req_def456",
      "amount": 80,
      "status": "paid"
    }
  ],
  "page": 1,
  "pageSize": 10
}
```

------------------------------------------------------------------------

## Webhook Endpoint

POST /webhooks/payfast

Receives payment confirmation events from PayFast.

### Example Payload

``` json
{
  "payment_id": "pf_123456",
  "payment_request_id": "req_abc123",
  "amount": 150.00,
  "status": "COMPLETE"
}
```

------------------------------------------------------------------------

# Error Responses

### 400 Bad Request

``` json
{
  "error": "VALIDATION_ERROR",
  "message": "Amount must be greater than zero"
}
```

### 401 Unauthorized

``` json
{
  "error": "UNAUTHORIZED",
  "message": "Authentication required"
}
```

### 404 Not Found

``` json
{
  "error": "NOT_FOUND",
  "message": "Payment request not found"
}
```

### 500 Internal Server Error

``` json
{
  "error": "SERVER_ERROR",
  "message": "Unexpected error occurred"
}
```

------------------------------------------------------------------------

# Data Entities

## User

-   id
-   name
-   email
-   phone
-   created_at

## PaymentRequest

-   id
-   user_id
-   amount
-   currency
-   description
-   status
-   payment_link
-   created_at
-   expires_at

## Payment

-   id
-   payment_request_id
-   provider
-   provider_reference
-   amount
-   status
-   paid_at

## WebhookEvent

-   id
-   provider
-   payload
-   received_at
