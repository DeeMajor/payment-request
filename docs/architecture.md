# PayLink Architecture Document

## 1. High-Level Architecture

PayLink is a mobile-first payment request platform. The system coordinates
payment requests between users and PayFast, without storing or moving funds.

### Evolutionary Architecture Strategy

This architecture is designed in two phases to avoid premature complexity:

- **Phase 1 (MVP)**: Single .NET API process handles everything — API
  requests, webhook processing, scheduled jobs (via hosted services), and
  QR generation. PostgreSQL is the only infrastructure dependency. No
  Redis, no message queue. This reduces deployment, monitoring, and
  operational overhead to near zero while proving the product.

- **Phase 2 (Scale)**: When traffic justifies it (sustained > 100 webhooks/min
  or > 1000 active users), introduce Redis (caching, rate limiting) and
  RabbitMQ (async webhook processing). The clean architecture boundaries
  make this a straightforward infrastructure swap — application and domain
  layers are unchanged.

The diagrams below show the full Phase 2 architecture. Phase 1 omits
the dashed-border components.

### System Components

```
+------------------+        +-------------------+        +------------------+
|                  |        |                   |        |                  |
|  React Frontend  +------->+  .NET Web API     +------->+  PostgreSQL      |
|  (Mobile-first)  |  HTTPS |  (Backend)        |        |  (Database)      |
|                  +<-------+                   +<-------+                  |
+------------------+        +--------+----------+        +------------------+
                                     |
                            +--------+----------+
                            |                   |
                            |  Background       |
                            |  Hosted Services  |
                            |  (in-process MVP) |
                            |                   |
                            +--------+----------+
                                     |
                    +----------------+----------------+
                    |                |                 |
             +------+-----+  +- - - - - - -+  +- - - - - - -+
             |            |  :             :  :             :
             |  PayFast   |  :  Redis      :  :  RabbitMQ   :
             |  Gateway   |  :  (Phase 2)  :  :  (Phase 2)  :
             |            |  :             :  :             :
             +------------+  +- - - - - - -+  +- - - - - - -+
```

### Component Interactions

```
User (Mobile Browser)
  |
  | HTTPS (TLS 1.3)
  v
React SPA (served via CDN)
  |
  | HTTPS REST API calls (JWT auth)
  v
.NET Web API (Stateless, containerized)
  |
  +---> PostgreSQL (persistent state, idempotency keys, rate limit counters)
  |
  +---> PayFast API (redirect URL generation)
  |
  +---> [Phase 2] Redis (caching, rate limiting)
  |
  +---> [Phase 2] RabbitMQ (async webhook processing)
  |
  v
Background Hosted Services (in-process, same deployment)
  +---> Sends notifications (SMS via provider)
  +---> Runs payment expiry checks (cron-style)
  +---> Runs reconciliation checks (cron-style)

PayFast
  |
  | HTTPS POST (webhook callback, form-encoded)
  v
.NET Web API  -->  /webhooks/payfast
```

### Network Boundary Diagram

```
+-----------------------------------------------------------------------+
|  CLOUD (Azure / AWS)                                                  |
|                                                                       |
|  +---------------------------+     +-------------------------------+  |
|  | PUBLIC SUBNET             |     | PRIVATE SUBNET                |  |
|  |                           |     |                               |  |
|  |  +---------------------+ |     |  +-------------------------+  |  |
|  |  | Load Balancer / CDN | |     |  | .NET API (containers)   |  |  |
|  |  | (TLS termination)   | +---->+  | - API instances (x N)   |  |  |
|  |  +---------------------+ |     |  +------------+------------+  |  |
|  |                           |     |               |               |  |
|  +---------------------------+     |  +------------+------------+  |  |
|                                    |  |                         |  |  |
|                                    |  |  PostgreSQL (managed)   |  |  |
|                                    |  |  [Phase 2] Redis        |  |  |
|                                    |  |  [Phase 2] RabbitMQ     |  |  |
|                                    |  |                         |  |  |
|                                    |  +-------------------------+  |  |
|                                    +-------------------------------+  |
+-----------------------------------------------------------------------+
```

---

## 2. Backend Architecture (Clean Architecture)

The backend follows Clean Architecture with four layers. Dependencies
point inward -- outer layers depend on inner layers, never the reverse.

```
+------------------------------------------------------------------+
|  Presentation Layer (API Controllers, Middleware, Filters)        |
+------------------------------------------------------------------+
|  Application Layer (Use Cases, DTOs, Interfaces, Validators)     |
+------------------------------------------------------------------+
|  Domain Layer (Entities, Value Objects, Domain Events, Rules)     |
+------------------------------------------------------------------+
|  Infrastructure Layer (EF Core, PayFast Client, Notifications)   |
+------------------------------------------------------------------+
```

### Project Structure

```
src/
  PayLink.Api/                  # Presentation Layer
    Controllers/
      PaymentRequestsController.cs
      WebhooksController.cs
      AuthController.cs
    Middleware/
      ExceptionHandlingMiddleware.cs
      RequestLoggingMiddleware.cs
      IdempotencyMiddleware.cs
      WebhookTimestampMiddleware.cs
    Filters/
      RateLimitingFilter.cs
    BackgroundJobs/              # Hosted services (in-process)
      ExpirePaymentRequestsJob.cs
      ReconciliationJob.cs
    Program.cs

  PayLink.Application/          # Application Layer
    PaymentRequests/
      Commands/
        CreatePaymentRequest/
          CreatePaymentRequestCommand.cs
          CreatePaymentRequestHandler.cs
          CreatePaymentRequestValidator.cs
        CancelPaymentRequest/
          ...
      Queries/
        GetPaymentRequest/
          GetPaymentRequestQuery.cs
          GetPaymentRequestHandler.cs
        ListPaymentRequests/
          ...
    Webhooks/
      Commands/
        ProcessWebhook/
          ProcessWebhookCommand.cs
          ProcessWebhookHandler.cs
    Notifications/
      Commands/
        SendPaymentNotification/
          ...
    Common/
      Interfaces/
        IPaymentGateway.cs
        INotificationService.cs
        IWebhookProcessor.cs       # Sync in MVP, queue-backed in Phase 2
      Behaviours/
        ValidationBehaviour.cs
        LoggingBehaviour.cs

  PayLink.Domain/               # Domain Layer
    Entities/
      PaymentRequest.cs
      Payment.cs
      User.cs
      WebhookEvent.cs
    ValueObjects/
      Money.cs
      Currency.cs               # Enum, not a string -- ZAR only for MVP
      PaymentStatus.cs
      PaymentRequestId.cs
    Events/
      PaymentCompletedEvent.cs
      PaymentRequestExpiredEvent.cs
    Exceptions/
      PaymentRequestExpiredException.cs
      DuplicateWebhookException.cs

  PayLink.Infrastructure/       # Infrastructure Layer
    Persistence/
      AppDbContext.cs
      Configurations/
        PaymentRequestConfiguration.cs
        PaymentConfiguration.cs
        UserConfiguration.cs
      Repositories/
        PaymentRequestRepository.cs
      Migrations/
    ExternalServices/
      PayFast/
        PayFastClient.cs
        PayFastSignatureValidator.cs
        PayFastIpAllowlist.cs
        PayFastFieldMapper.cs   # Maps PayFast field names to domain model
      Notifications/
        SmsSender.cs            # MVP: SMS only (target market uses WhatsApp/SMS)
    Deployment/
      Dockerfile
      docker-compose.yml
      deploy.sh
```

### Layer Responsibilities

| Layer              | Responsibility                                            | Depends On  |
| ------------------ | --------------------------------------------------------- | ----------- |
| **Presentation**   | HTTP handling, request/response mapping, middleware        | Application |
| **Application**    | Orchestrates use cases, defines interfaces, validation     | Domain      |
| **Domain**         | Business rules, entities, value objects, domain events     | Nothing     |
| **Infrastructure** | Database access, external APIs, notifications              | Application |

### Where Business Logic Lives

- **Domain Layer**: Core rules that are always true regardless of how the
  system is called. Examples: a payment request cannot be paid after expiry,
  amount must be positive, status transitions follow defined rules,
  currency must be ZAR.

- **Application Layer**: Orchestration logic that coordinates domain objects
  and infrastructure. Examples: create a payment request then generate a
  QR code, process a webhook then trigger a notification.

The Presentation and Infrastructure layers contain zero business logic.

---

## 3. Payment Flow

### 3.1 Payment Request Creation

```
User                React App           API                  Database         PayFast
 |                    |                  |                      |                |
 |  Fill form         |                  |                      |                |
 +--amount, desc----->|                  |                      |                |
 |                    |  POST /payment-  |                      |                |
 |                    |  requests        |                      |                |
 |                    +----------------->|                      |                |
 |                    |                  |  Validate input      |                |
 |                    |                  |  Enforce currency=ZAR|                |
 |                    |                  |  Create entity       |                |
 |                    |                  |  Generate ID         |                |
 |                    |                  +--------------------->|                |
 |                    |                  |  INSERT              |                |
 |                    |                  |<---------------------+                |
 |                    |                  |                      |                |
 |                    |                  |  Build PayFast URL   |                |
 |                    |                  |  (using PayFast      |                |
 |                    |                  |   field names:       |                |
 |                    |                  |   merchant_id,       |                |
 |                    |                  |   amount, item_name) |                |
 |                    |                  +------------------------------------->|
 |                    |                  |  Return redirect URL |                |
 |                    |                  |<-------------------------------------+
 |                    |                  |                      |                |
 |                    |                  |  Generate QR code    |                |
 |                    |                  |  (on-the-fly, in     |                |
 |                    |                  |   memory, return as  |                |
 |                    |                  |   base64 data URI)   |                |
 |                    |                  |                      |                |
 |                    |  Response:       |                      |                |
 |                    |  id, link, qr    |                      |                |
 |                    |<-----------------+                      |                |
 |  Show QR + link    |                  |                      |                |
 |<-------------------+                  |                      |                |
```

**QR code strategy**: Generated on-the-fly using a library like
`QRCoder`. The QR encodes the payment link URL. Returned as a base64
data URI in the API response. No storage required -- regenerate on
subsequent `GET` requests. This keeps infrastructure simple and avoids
blob storage for MVP.

### 3.2 Customer Payment

```
Customer            Browser             API                  PayFast
 |                    |                   |                     |
 |  Scan QR / tap     |                   |                     |
 |  payment link      |                   |                     |
 +--GET /pay/{id}---->|                   |                     |
 |                    +--GET /pay/{id}--->|                     |
 |                    |                   |  Lookup request     |
 |                    |                   |  Validate active    |
 |                    |                   |  (not expired,      |
 |                    |                   |   not already paid) |
 |                    |                   |                     |
 |                    |  302 Redirect to  |                     |
 |                    |  PayFast checkout |                     |
 |                    |<------------------+                     |
 |                    |                   |                     |
 |                    +--PayFast page---->|                     |
 |                    |                   |                     |
 |  Customer pays     |                   |                     |
 |  on PayFast page   |                   |                     |
 +--card/eft/etc----->+                   |                     |
 |                    +--payment--------->|                     |
 |                    |                   |  Process payment    |
 |                    |                   |                     |
 |                    |  Return to        |                     |
 |                    |  return_url       |                     |
 |                    |<------------------+                     |
 |  See confirmation  |                   |                     |
 |<-------------------+                   |                     |
```

### 3.3 Webhook Confirmation

PayFast sends webhooks as `application/x-www-form-urlencoded` POST
requests (not JSON). The actual PayFast field names are used below.

```
PayFast              API                  Database
 |                    |                     |
 |  POST /webhooks/   |                     |
 |  payfast            |                     |
 |  (form-encoded)    |                     |
 +--ITN payload------>|                     |
 |                    |                     |
 |                    |  1. Validate source |
 |                    |     IP against      |
 |                    |     PayFast         |
 |                    |     allowlist       |
 |                    |                     |
 |                    |  2. Validate        |
 |                    |     signature       |
 |                    |     (MD5 -- PayFast |
 |                    |      requirement,   |
 |                    |      see security   |
 |                    |      notes below)   |
 |                    |                     |
 |                    |  3. Validate        |
 |                    |     timestamp       |
 |                    |     (reject if      |
 |                    |      > 5 min old)   |
 |                    |                     |
 |                    |  4. Check           |
 |                    |     idempotency     |
 |                    |     (pf_payment_id) |
 |                    |                     |
 |                    |  5. Validate amount |
 |                    |     amount_gross    |
 |                    |     matches stored  |
 |                    |     request amount  |
 |                    |                     |
 |                    |  BEGIN TRANSACTION  |
 |                    |                     |
 |                    |  6. Store raw       |
 |                    |     webhook event   |
 |                    +--INSERT webhook---->|
 |                    |<--------------------+
 |                    |                     |
 |                    |  7. Load payment    |
 |                    |     request (lock)  |
 |                    +--SELECT FOR UPDATE->|
 |                    |<--------------------+
 |                    |                     |
 |                    |  8. Update status   |
 |                    |     pending -> paid |
 |                    +--UPDATE------------>|
 |                    |<--------------------+
 |                    |                     |
 |                    |  COMMIT             |
 |                    |                     |
 |  200 OK            |                     |
 |<-------------------+                     |
 |                    |                     |
 |                    |  9. Send            |
 |                    |     notification    |
 |                    |     (after commit,  |
 |                    |      fire-and-      |
 |                    |      forget)        |
```

**MVP note**: In Phase 1, webhook processing happens synchronously in the
API request. The endpoint validates, stores, processes, and returns 200.
Notifications are sent after the transaction commits (fire-and-forget).
In Phase 2, steps 6-9 are offloaded to a queue consumer for faster
webhook response times.

### Step-by-Step Summary

1. **User creates payment request** via the API. System generates a unique
   ID, stores the request, and returns a payment link + QR code.

2. **Customer opens the link** or scans the QR code. The API validates the
   request is still active (not expired, not already paid), then redirects
   the customer to PayFast's hosted checkout page.

3. **Customer pays on PayFast**. PayFast handles the actual payment
   processing (card, EFT, etc.). PayLink never touches card details.

4. **PayFast sends an ITN (Instant Transaction Notification)** to
   `POST /webhooks/payfast` as a form-encoded POST. The API validates
   the source (IP allowlist + signature + timestamp), checks for duplicate
   delivery, validates the amount matches, processes the status update in
   a transaction, and responds `200 OK`.

5. **Notification fires** after the transaction commits. SMS sent to the
   user confirming payment was received.

---

## 4. Security Design

### 4.1 Authentication and Authorization

```
+-------------------+       +------------------+       +-----------------+
|  React App        |       |  .NET API        |       |  PostgreSQL     |
|                   |       |                  |       |                 |
|  Login form       +------>+  /auth/login     |       |                 |
|                   |       |  Issue JWT       |       |                 |
|  Store token      |       |  (short-lived)   |       |                 |
|  in memory only   |       |  + refresh token |       |                 |
|                   |       |  (HttpOnly       |       |                 |
|  Attach JWT to    |       |   cookie)        |       |                 |
|  API requests     +------>+                  |       |                 |
|  via Auth header  |       |  Validate JWT    |       |                 |
|                   |       |  on each request |       |                 |
|                   |       |  Check user owns |       |                 |
|                   |       |  the resource    +------>+  Row-level      |
|                   |       |                  |       |  filtering by   |
|                   |       |                  |       |  user_id        |
+-------------------+       +------------------+       +-----------------+
```

| Mechanism             | Implementation                                          |
| --------------------- | ------------------------------------------------------- |
| **Authentication**    | JWT (RS256, 15-min expiry) + refresh tokens             |
| **Token storage**     | Access token in memory, refresh token as HttpOnly cookie |
| **Authorization**     | Resource ownership check: `WHERE user_id = @currentUser`|
| **Password storage**  | bcrypt with cost factor 12                              |
| **Rate limiting**     | Token bucket algorithm (see 4.5 below)                  |
| **CORS**              | Restricted to known frontend origins                    |

### 4.2 Data Protection

| Concern                 | Approach                                              |
| ----------------------- | ----------------------------------------------------- |
| **Data in transit**     | TLS 1.3 on all connections (API, DB)                  |
| **Data at rest**        | PostgreSQL Transparent Data Encryption (TDE)          |
| **PII fields**          | Encrypt email/phone at application level (AES-256)    |
| **Secrets management**  | Azure Key Vault / AWS Secrets Manager, never in code  |
| **Database credentials**| Rotated automatically via secrets manager             |
| **Audit logging**       | All state-changing operations logged with actor + IP  |
| **Soft delete**         | Users and payment requests use `deleted_at` timestamp (see 4.6) |

### 4.3 Webhook Validation (PayFast)

Every incoming webhook goes through a multi-step validation pipeline
before any processing occurs:

```
Incoming Webhook (form-encoded ITN)
      |
      v
+-----+-------+     FAIL     +------------------+
| 1. IP Allow  +------------->  403 Forbidden    |
|    list      |              +------------------+
+-----+-------+
      | PASS
      v
+-----+-------+     FAIL     +------------------+
| 2. Signature +------------->  401 Unauthorized |
|    check     |              +------------------+
+-----+-------+
      | PASS
      v
+-----+---------+    FAIL    +------------------+
| 3. Timestamp   +----------->  400 Stale Event  |
|    window      |            +------------------+
|    (< 5 min)   |
+-----+---------+
      | PASS
      v
+-----+-------+     DUPLICATE +------------------+
| 4. Idempotency+------------->  200 OK (no-op)  |
|    check      |              +------------------+
+-----+---------+
      | NEW
      v
+-----+-------+     FAIL     +------------------+
| 5. Amount    +------------->  400 Bad Request  |
|    match     |              +------------------+
+-----+-------+
      | PASS
      v
  Store + Process (in transaction)
```

**Signature validation** (PayFast ITN-specific):

PayFast uses MD5 for signature generation. This is a PayFast platform
constraint, not a choice. MD5 is cryptographically weak for collision
resistance, which is why the IP allowlist is the primary trust
mechanism, not the signature. The signature provides tamper detection
for payloads originating from verified PayFast IPs.

Validation steps:
1. Collect all POST parameters except `signature`.
2. Sort alphabetically by key.
3. URL-encode each value and concatenate as `key=value&key=value`.
4. Append the PayFast passphrase: `&passphrase={passphrase}`.
5. MD5 hash the resulting string.
6. Compare with the received `signature` parameter.

**IP allowlist**: PayFast publishes known server IPs for ITN delivery.
Only accept webhooks from these addresses. Behind a load balancer,
validate `X-Forwarded-For` but only trust the header when the immediate
connection comes from the load balancer's known IP.

**Timestamp validation**: Reject webhook events where the timestamp
is older than 5 minutes. This prevents replay attacks where a captured
webhook payload is resent later. Combined with idempotency (step 4),
this closes the replay window even if an attacker obtains a valid
signed payload.

### 4.4 PayFast Field Mapping

PayFast sends specific field names that differ from the domain model.
The `PayFastFieldMapper` handles this translation:

| PayFast Field       | Domain Field            | Notes                         |
| ------------------- | ----------------------- | ----------------------------- |
| `pf_payment_id`     | `provider_reference`    | Used as idempotency key       |
| `payment_status`    | `status`                | "COMPLETE" -> `Paid`          |
| `amount_gross`      | `amount`                | Compared to request amount    |
| `amount_fee`        | `provider_fee`          | PayFast processing fee        |
| `amount_net`        | `net_amount`            | Amount after fees             |
| `custom_str1`       | `payment_request_id`    | Links back to our request     |
| `merchant_id`       | _(validated)_           | Must match our merchant ID    |
| `item_name`         | `description`           | Payment description           |
| `signature`         | _(validated)_           | MD5 signature for validation  |

The API design document uses simplified field names. The infrastructure
layer handles the mapping between PayFast's actual field names and the
domain model.

### 4.5 Rate Limiting Design

Rate limiting uses the **token bucket algorithm**, implemented in
PostgreSQL for MVP (no Redis dependency) and optionally backed by Redis
in Phase 2.

```
MVP implementation (PostgreSQL-backed):
  Table: rate_limits (key TEXT PK, tokens INT, last_refill TIMESTAMPTZ)

Phase 2 (Redis-backed):
  Key: "ratelimit:{key}"
  Uses Redis MULTI/EXEC for atomic token decrement

Rate limit tiers:
+---------------------------+--------+-----------+---------------------------+
| Endpoint                  | Window | Max Reqs  | Key                       |
+---------------------------+--------+-----------+---------------------------+
| POST /auth/login          | 15 min | 5         | ip:{remote_ip}            |
| POST /auth/register       | 1 hour | 3         | ip:{remote_ip}            |
| POST /payment-requests    | 1 min  | 30        | user:{user_id}            |
| GET  /payment-requests    | 1 min  | 60        | user:{user_id}            |
| POST /webhooks/payfast    | 1 min  | 100       | ip:{remote_ip}            |
+---------------------------+--------+-----------+---------------------------+

Response when limited:
  HTTP 429 Too Many Requests
  Headers: Retry-After: {seconds}
           X-RateLimit-Limit: {max}
           X-RateLimit-Remaining: {remaining}
           X-RateLimit-Reset: {unix_timestamp}
```

The load balancer also enforces a global per-IP rate limit as a first
line of defence before requests reach the application.

### 4.6 Soft Delete and Data Retention

Financial records must be retained even when users delete their accounts.
The system uses soft deletes:

```
Users:
  - deleted_at: TIMESTAMPTZ (NULL = active)
  - On "delete account": set deleted_at, anonymize PII (name, email, phone)
  - Payment requests and payments are retained (orphaned but intact)
  - Anonymized records kept for 7 years (financial compliance)

Payment Requests:
  - deleted_at: TIMESTAMPTZ (NULL = active)
  - Only "pending" requests can be soft-deleted (user cancellation)
  - "paid" requests are never deleted (audit trail)
  - Soft-deleted requests excluded from list queries via default scope
```

### 4.7 Attack Prevention

| Attack                  | Mitigation                                              |
| ----------------------- | ------------------------------------------------------- |
| **Replay attacks**      | Timestamp window (5 min) + idempotency key (`pf_payment_id`) |
| **Payload tampering**   | Signature validation on every webhook                   |
| **Amount manipulation** | Verify `amount_gross` matches stored payment request amount |
| **SQL injection**       | Parameterized queries via EF Core; no raw SQL           |
| **XSS**                 | React auto-escapes output; CSP headers enforced         |
| **CSRF**                | SameSite cookies; no cookie-based auth for API calls    |
| **Brute force**         | Rate limiting on auth endpoints (5 attempts/15 min)     |
| **Enumeration**         | Opaque IDs (`req_` prefix + random); generic 404s       |
| **Man-in-the-middle**   | TLS everywhere; HSTS header with preload                |

---

## 5. Data Consistency and Reliability

### 5.1 Data Model Additions

All entities include audit timestamps:

```sql
-- Every table includes these columns
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()  -- updated via trigger

-- Trigger for automatic updated_at
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Applied to each table
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON payment_requests
  FOR EACH ROW EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON payments
  FOR EACH ROW EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_timestamp();
```

Currency is enforced as an enum, not a freeform string:

```sql
CREATE TYPE currency AS ENUM ('ZAR');

ALTER TABLE payment_requests
  ADD CONSTRAINT chk_currency CHECK (currency = 'ZAR');
```

When multi-currency is needed in the future, add new values to the enum
and update the domain validation. This prevents invalid currency values
at the database level.

### 5.2 Idempotency Handling

There are two categories of idempotency in this system:

**API-level idempotency** (client-originated requests):

```
Client sends:  POST /payment-requests
               Header: Idempotency-Key: "client-generated-uuid"

MVP (PostgreSQL-backed):
  Table: idempotency_keys (
    key         TEXT PRIMARY KEY,
    response    JSONB NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
  )

Server logic:
  1. Check idempotency_keys table for key
  2. If found  -> return cached response (200)
  3. If not    -> process request
  4. Store response in idempotency_keys
  5. Return response
  6. Cleanup: delete keys older than 24 hours (scheduled job)

Phase 2 (Redis-backed):
  Key: "idempotency:{key}"
  TTL: 24 hours (automatic cleanup)
```

**Webhook-level idempotency** (PayFast-originated events):

```
PayFast sends: POST /webhooks/payfast
               Body includes: pf_payment_id=123456

Server logic:
  1. Check webhook_events table for provider_reference = "123456"
  2. If found  -> return 200 OK (no-op, already processed)
  3. If not    -> store event, process, return 200 OK
```

### 5.3 Webhook Retry Handling

PayFast retries failed ITN deliveries. The system must handle this
gracefully:

```
+------------------+---------------------------------------------------+
| Scenario         | System Behaviour                                  |
+------------------+---------------------------------------------------+
| First delivery   | Store event, process normally, return 200          |
| Retry (duplicate)| Detect via pf_payment_id, return 200 (no reprocess)|
| Malformed payload| Return 400, PayFast may retry                     |
| Server error     | Return 500, PayFast will retry                    |
| Server down      | PayFast retries with backoff; reconciliation job   |
|                  | catches missed events                              |
| Stale timestamp  | Return 400, event is too old (> 5 min)            |
+------------------+---------------------------------------------------+
```

In the MVP, webhook processing is synchronous but fast (single DB
transaction). The endpoint should respond well within PayFast's
timeout window. In Phase 2, processing moves to a queue consumer
for sub-100ms webhook response times.

### 5.4 Transaction Boundaries

Payment status updates use database transactions to ensure consistency:

```csharp
// In the webhook processing handler
await using var transaction = await _dbContext.Database
    .BeginTransactionAsync(IsolationLevel.ReadCommitted);

try
{
    // 1. Load payment request (with row lock)
    var request = await _dbContext.PaymentRequests
        .FromSqlRaw(
            "SELECT * FROM payment_requests WHERE id = {0} FOR UPDATE",
            paymentRequestId)
        .FirstOrDefaultAsync();

    // 2. Validate state transition
    if (request.Status != PaymentStatus.Pending)
        return; // Already processed -- idempotent

    // 3. Validate amount matches
    if (webhookAmountGross != request.Amount)
        throw new AmountMismatchException(
            request.Id, request.Amount, webhookAmountGross);

    // 4. Create payment record
    var payment = Payment.Create(
        request.Id,
        webhookPfPaymentId,  // PayFast's pf_payment_id
        amountGross,
        amountFee,
        amountNet);
    _dbContext.Payments.Add(payment);

    // 5. Update status
    request.MarkAsPaid(payment);

    // 6. Mark webhook event as processed
    webhookEvent.MarkProcessed();

    await _dbContext.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}

// 7. Notification sent AFTER commit (fire-and-forget, outside transaction)
await _notificationService.SendPaymentReceivedAsync(request.UserId, request.Id);
```

Key principles:
- Use `SELECT ... FOR UPDATE` to prevent concurrent processing of the
  same payment request.
- Validate the status transition before applying it (reject if already paid).
- Keep transactions short -- only the database write, not external calls.
- Notifications are sent after the transaction commits, not inside it.
  If notification delivery fails, the payment is still recorded correctly.

---

## 6. Scalability Considerations

### 6.1 Horizontal Scaling Strategy

```
                    +-------------------+
                    |  Load Balancer    |
                    |  (health checks)  |
                    +---+-------+---+---+
                        |       |   |
               +--------+   +--+   +--------+
               |             |              |
        +------+------+ +---+---+  +-------+------+
        | API          | | API   |  | API          |
        | Instance 1   | | Inst 2|  | Instance N   |
        +--------------+ +-------+  +--------------+
```

| Component     | MVP                                      | Phase 2                                       |
| ------------- | ---------------------------------------- | --------------------------------------------- |
| **API**       | Single container, hosted services inside  | Horizontal replicas behind load balancer       |
| **Workers**   | In-process hosted services               | Separate containers, competing queue consumers |
| **PostgreSQL**| Single managed instance + connection pool | Read replicas for query scaling                |
| **Redis**     | Not used                                 | Managed cluster (ElastiCache / Azure Cache)    |
| **RabbitMQ**  | Not used                                 | Managed service or clustered deployment        |
| **Frontend**  | CDN-served static assets                 | CDN-served static assets (infinite scale)      |

### 6.2 Stateless Services

The API is fully stateless. All state lives in PostgreSQL.

Checklist for statelessness:
- No in-memory session state (JWT is self-contained)
- No local file storage (QR codes generated on-the-fly, returned as
  base64 data URIs -- no blob storage needed)
- No sticky sessions required
- Configuration loaded from environment variables / secrets manager
- Any instance can handle any request

### 6.3 Database Scaling

**Phase 1 (MVP)**: Single PostgreSQL instance with connection pooling
(built-in Npgsql pooling, max 20 connections).

**Phase 2 (Growth)**: Add PgBouncer for connection pooling across
multiple API instances. Add read replicas. Route read queries
(`GET /payment-requests`, list queries) to replicas. Write queries
stay on the primary.

**Phase 3 (Scale)**: If needed, partition the `payment_requests` and
`payments` tables by `created_at` (range partitioning). Older data
moves to cheaper storage.

**Indexing strategy**:
```sql
-- Primary lookups
CREATE INDEX idx_payment_requests_user_id ON payment_requests(user_id);
CREATE INDEX idx_payment_requests_status ON payment_requests(status)
    WHERE status = 'pending';

-- Webhook processing
CREATE UNIQUE INDEX idx_webhook_events_provider_ref
    ON webhook_events(provider, provider_reference);

-- Expiry processing
CREATE INDEX idx_payment_requests_expires_at ON payment_requests(expires_at)
    WHERE status = 'pending';

-- Soft delete filtering
CREATE INDEX idx_payment_requests_active
    ON payment_requests(user_id, created_at)
    WHERE deleted_at IS NULL;

CREATE INDEX idx_users_active ON users(id) WHERE deleted_at IS NULL;

-- Idempotency cleanup
CREATE INDEX idx_idempotency_keys_created_at ON idempotency_keys(created_at);
```

---

## 7. Failure Scenarios

### 7.1 Payment Succeeds but Webhook Fails

```
Problem:  Customer paid on PayFast, but the webhook never arrives
          (network issue, server downtime, etc.)

Solution: Scheduled reconciliation job (hosted service)

The reconciliation job runs as a .NET BackgroundService / IHostedService
inside the API process. It uses PayFast's transaction query API to check
the status of pending payment requests.

Approach:
  1. Every 15 minutes, query payment_requests WHERE status = 'pending'
     AND created_at < NOW() - INTERVAL '30 minutes'
  2. For each result, call PayFast's ad-hoc transaction query endpoint:
     POST https://api.payfast.co.za/process/query/{token}
  3. If PayFast reports the transaction as COMPLETE but our record is
     still pending, process it as if the webhook arrived (same handler,
     same transaction boundaries, same idempotency checks)
  4. Log a warning: "Reconciliation caught missed webhook" with details

Note: PayFast's query API requires the original payment token. Store
the token at payment request creation time for later reconciliation.
If PayFast does not support this query for your integration tier,
fall back to manual reconciliation alerts.
```

### 7.2 Duplicate Webhook Events

```
Problem:  PayFast sends the same ITN multiple times (retry logic)

Solution: Idempotency via unique constraint

  ITN arrives with pf_payment_id = "123456"
      |
      v
  Check: SELECT 1 FROM webhook_events
         WHERE provider = 'payfast'
         AND provider_reference = '123456'
      |
      +-- EXISTS? --> Return 200 OK, skip processing
      |
      +-- NOT EXISTS? --> INSERT event, process normally
```

The `UNIQUE INDEX` on `(provider, provider_reference)` acts as a database-
level guarantee against duplicates, even under concurrent requests. If two
identical webhooks arrive at the exact same time, one INSERT will succeed
and the other will fail with a unique constraint violation -- caught and
handled as a duplicate.

### 7.3 Network Failures

| Failure Point                     | Handling                                    |
| --------------------------------- | ------------------------------------------- |
| API cannot reach PostgreSQL       | Connection pool retry (3 attempts, exponential backoff). Return 503 to client. Health check fails, load balancer stops routing. |
| API cannot reach PayFast          | Return 502 to client with message to retry. Payment request still created in DB (can retry URL generation). |
| Frontend cannot reach API         | Client-side retry with exponential backoff. Show user-friendly error after 3 attempts. |
| Notification delivery fails       | Log error, do not fail the payment processing. Payment is already committed. User can check status manually. Reconciliation job can re-trigger notification for unacknowledged payments. |

### 7.4 Gateway Delays

PayFast may take longer than expected to process payments or send webhooks:

- **Customer-facing**: Show a "payment processing" state in the frontend.
  Poll `GET /payment-requests/{id}` with exponential backoff (2s, 4s, 8s)
  up to 60 seconds. If still pending, show "We'll notify you when confirmed."

- **System-side**: Do not timeout payment requests prematurely. The expiry
  on a payment request is set by the user (hours/days, not seconds). The
  reconciliation job handles delayed webhooks.

---

## 8. Observability

### 8.1 Logging Strategy

Use structured logging (Serilog) with correlation IDs for request tracing.

```
Log Levels:
  Information  - Request received, payment created, webhook processed
  Warning      - Duplicate webhook, rate limit hit, slow query,
                 reconciliation caught a missed webhook
  Error        - Unhandled exception, webhook validation failure,
                 notification delivery failure, amount mismatch
  Critical     - Database unreachable
```

**What to log** (with examples):

```json
{
  "timestamp": "2026-03-29T14:00:00Z",
  "level": "Information",
  "correlationId": "corr_abc123",
  "message": "Payment request created",
  "userId": "usr_def456",
  "paymentRequestId": "req_ghi789",
  "amount": 150.00,
  "currency": "ZAR",
  "duration_ms": 45
}
```

```json
{
  "timestamp": "2026-03-29T14:01:00Z",
  "level": "Warning",
  "message": "Duplicate webhook received",
  "provider": "payfast",
  "providerReference": "pf_123456",
  "webhookEventId": "evt_existing_id"
}
```

```json
{
  "timestamp": "2026-03-29T14:02:00Z",
  "level": "Warning",
  "message": "Reconciliation caught missed webhook",
  "paymentRequestId": "req_abc789",
  "payfastStatus": "COMPLETE",
  "localStatus": "pending",
  "pendingSince": "2026-03-29T13:15:00Z"
}
```

**What NOT to log**: Card numbers, bank details, full request bodies
containing PII, authentication tokens, passphrase values.

**Log destination**: Centralized log aggregation (Azure Monitor / AWS
CloudWatch / ELK stack). Retain 90 days hot, 1 year cold storage.

### 8.2 Metrics

Expose metrics via Prometheus endpoint (`/metrics`) or Application Insights:

| Metric                              | Type      | Alert Threshold         |
| ----------------------------------- | --------- | ----------------------- |
| `http_requests_total`               | Counter   | -                       |
| `http_request_duration_seconds`     | Histogram | p99 > 2s                |
| `payment_requests_created_total`    | Counter   | -                       |
| `payments_completed_total`          | Counter   | -                       |
| `webhook_received_total`            | Counter   | -                       |
| `webhook_validation_failures_total` | Counter   | > 10/hour               |
| `webhook_processing_duration`       | Histogram | p99 > 5s                |
| `webhook_duplicate_total`           | Counter   | > 50/hour               |
| `db_connection_pool_active`         | Gauge     | > 80% of max            |
| `payment_reconciliation_mismatches` | Counter   | > 0 (always alert)      |
| `notification_delivery_failures`    | Counter   | > 5/hour                |
| `rate_limit_rejections_total`       | Counter   | -                       |

### 8.3 Alerting

```
CRITICAL (page oncall):
  - API error rate > 5% for 5 minutes
  - Webhook processing failure rate > 10%
  - Database connection failures
  - Reconciliation finds mismatches (money discrepancy)
  - Amount mismatch on webhook (possible tampering)

WARNING (Slack/email notification):
  - p99 latency > 2 seconds
  - Webhook signature validation failures spike
  - Payment request creation rate drops > 50% vs same hour yesterday
  - Disk usage > 80%
  - Notification delivery failure rate > 20%

INFO (dashboard only):
  - Daily payment volume
  - User registration rate
  - Average payment amount
  - Reconciliation job run count
```

### 8.4 Health Checks

Health checks distinguish between critical and degraded dependencies:

```
GET /health/live
  -> 200 if process is running (k8s liveness)
  -> Checks nothing external

GET /health/ready
  -> 200 if PostgreSQL is reachable (k8s readiness)
  -> Only checks critical dependencies
  -> Does NOT check Redis/Queue (degraded != unhealthy)

GET /health/startup
  -> 200 after migrations complete (k8s startup)

Response body (for /health/ready):
{
  "status": "healthy",
  "checks": {
    "postgresql": "healthy",
    "redis": "degraded",       // Phase 2 only
    "rabbitmq": "degraded"     // Phase 2 only
  }
}
```

Non-critical dependencies (Redis, RabbitMQ in Phase 2) report their
status but do not cause the readiness check to fail. This prevents
a Redis outage from taking down the entire API when the system can
still function without caching.

---

## 9. Deployment and Operations

### 9.1 Deployment Strategy

```
MVP Deployment (single container):

  +--------------------------------------------------+
  | Docker Container                                  |
  |                                                   |
  |  .NET Web API                                     |
  |   +-- API Controllers                             |
  |   +-- Background Hosted Services                  |
  |       +-- ExpirePaymentRequestsJob                |
  |       +-- ReconciliationJob                        |
  |                                                   |
  +--------------------------------------------------+
       |
       v
  PostgreSQL (managed: Azure Database / AWS RDS)
```

**Deployment process** (zero-downtime rolling update):

```
1. Build new Docker image, tag with git SHA
2. Run database migrations (must be backward-compatible)
   - Add new columns as nullable first
   - Deploy code that handles both old and new schema
   - Backfill data if needed
   - In next release: add NOT NULL constraint
3. Rolling update: deploy new container alongside old
   - Health check must pass before old container is removed
   - Load balancer drains connections from old instance
4. Verify: check health endpoints + key metrics for 5 minutes
5. If metrics degrade: roll back to previous image tag
```

**Database migration safety rules**:
- Never drop a column in the same deploy that stops using it
- Never rename a column -- add new, migrate data, drop old
- Always test migrations against a copy of production data
- Migrations run automatically on startup (EF Core)

### 9.2 Backup and Recovery

| Concern       | Strategy                                                   |
| ------------- | ---------------------------------------------------------- |
| **RPO**       | 5 minutes (point-in-time recovery via WAL archiving)       |
| **RTO**       | 30 minutes (restore from latest snapshot + replay WAL)     |
| **Backups**   | Automated daily snapshots (managed DB), retained 30 days   |
| **WAL**       | Continuous archiving to object storage (S3/Azure Blob)     |
| **Testing**   | Monthly restore drill to verify backup integrity           |
| **Secrets**   | Secrets manager has its own backup/versioning              |

For MVP, use the managed database service's built-in backup (Azure
Database for PostgreSQL or AWS RDS). Both provide automated backups
with point-in-time recovery out of the box.

---

## 10. Optional Enhancements (Phase 2)

These components are introduced only when MVP traffic justifies the
operational complexity.

### 10.1 Message Queue (RabbitMQ)

Trigger: webhook processing latency causes PayFast timeout retries, or
sustained webhook volume exceeds 100/minute.

```
Exchanges and Queues:
  paylink.webhooks.received    -> webhook processing worker
  paylink.notifications.send   -> notification delivery worker

Dead Letter Queue:
  paylink.dlq                  -> failed messages after 3 retries
                                  (manual review + alerting)
```

Migration path from MVP:
1. Deploy RabbitMQ (managed service).
2. Swap `IWebhookProcessor` implementation from `SyncWebhookProcessor`
   to `QueuedWebhookProcessor`. No application or domain layer changes.
3. Deploy separate worker container that consumes from the queue.
4. Webhook endpoint now just validates + stores + enqueues (< 100ms).

### 10.2 Redis Cache

Trigger: database connection pool saturation on read-heavy queries,
or rate limiting needs sub-millisecond response times.

```
Redis Cache Layers:

1. Payment Request (read-heavy):
   Key:    "pr:{id}"
   TTL:    5 minutes
   Invalidate: on status change

2. Rate Limiting:
   Key:    "ratelimit:{key}"
   TTL:    sliding window

3. Idempotency:
   Key:    "idempotency:{key}"
   TTL:    24 hours
   Value:  serialized response
```

Migration path from MVP:
1. Deploy Redis (managed service).
2. Register Redis-backed implementations via DI.
3. PostgreSQL-backed rate limiting and idempotency tables become fallbacks.

### 10.3 Notification Channels

MVP ships with SMS only (target users share payment links via WhatsApp
and SMS). Future channels and their implementation:

| Channel      | Provider          | Trigger                      | Fallback      |
| ------------ | ----------------- | ---------------------------- | ------------- |
| **SMS**      | Twilio / Africa's Talking | Payment confirmed     | Log + retry   |
| **Email**    | SendGrid          | Payment confirmed (if email set) | SMS        |
| **Push**     | Firebase (FCM)    | Payment confirmed (if app installed) | SMS   |
| **WhatsApp** | WhatsApp Business API | Payment confirmed         | SMS           |

Notification delivery is always fire-and-forget relative to the payment
transaction. A failed notification never blocks or rolls back a payment
confirmation.

### 10.4 API Keys for Integrations

When third parties want to integrate PayLink into their own systems
(e.g., a POS system), add API key authentication:

```
Table: api_keys (
  id          UUID PRIMARY KEY,
  user_id     UUID REFERENCES users(id),
  key_hash    TEXT NOT NULL,          -- bcrypt hash of the key
  prefix      TEXT NOT NULL,          -- first 8 chars for identification
  name        TEXT NOT NULL,          -- "My POS system"
  scopes      TEXT[] NOT NULL,        -- {"payment_requests:write", "payment_requests:read"}
  last_used   TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL,
  revoked_at  TIMESTAMPTZ            -- NULL = active
)

Usage:
  Header: Authorization: Bearer pk_live_abc123...

  - API keys are an alternative to JWT for machine-to-machine access
  - Keys are shown once at creation, stored as bcrypt hash
  - Scoped to specific permissions
  - Rate limited separately from user JWT requests
```

This is not needed for MVP but the auth middleware should be designed
to accept both JWT and API key authentication from the start, so
adding API keys later is a configuration change, not a rewrite.

---

## Summary

This architecture prioritizes:

1. **Right-sized for MVP**: PostgreSQL-only infrastructure with in-process
   hosted services. No Redis, no message queue, no separate worker
   deployments. Complexity is added only when traffic demands it.

2. **Security**: Multi-layer webhook validation (IP allowlist as primary
   trust, MD5 signature as tamper detection, timestamp window for replay
   prevention), encryption at rest and in transit, JWT auth, rate limiting.

3. **Reliability**: Idempotent operations, transactional consistency,
   reconciliation jobs for missed webhooks, soft deletes for financial
   data retention, and defined backup/recovery targets (RPO: 5 min,
   RTO: 30 min).

4. **PayFast accuracy**: Real PayFast field names (`pf_payment_id`,
   `amount_gross`, `payment_status`), form-encoded ITN format, MD5
   signature process, and explicit mapping between PayFast fields and
   domain model.

5. **Observability**: Structured logging, Prometheus metrics, health
   checks that distinguish critical vs. degraded dependencies, and
   tiered alerting.

6. **Clear scaling path**: Phase 1 (single process + PostgreSQL) ->
   Phase 2 (Redis + RabbitMQ + separate workers) with explicit
   migration steps and triggers for when to transition.

7. **Maintainability**: Clean Architecture with clear layer boundaries,
   CQRS-style command/query separation, and infrastructure interfaces
   that allow swapping implementations without touching business logic.
