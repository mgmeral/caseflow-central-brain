# CaseFlow — Frontend Integration Contract

**Source of truth:** controllers + DTOs in `src/main/java/com/caseflow/`.
**TypeScript types:** `docs/api-models.ts`
**Swagger UI:** `http://localhost:8080/swagger-ui.html`

> This document is the frozen integration contract for the frontend.
> All shapes are derived from actual Java code, not assumptions.
> When in doubt, verify against Swagger UI or the controller source.

---

## Base URL

```
http://localhost:8080/api
```

---

## Auth Flow

CaseFlow uses **JWT Bearer** tokens. No session cookie.

### Login

```
POST /api/auth/login
Content-Type: application/json

{ "username": "alice", "password": "admin123" }
```

Response `200`:
```json
{
  "accessToken":  "eyJ...",
  "refreshToken": "550e8400-e29b-41d4-a716-...",
  "expiresIn":    3600,
  "tokenType":    "Bearer"
}
```

> `expiresIn` is in **seconds** (not milliseconds). Store both tokens.

### Authenticated requests

```
Authorization: Bearer <accessToken>
```

### Token refresh

```
POST /api/auth/refresh
{ "refreshToken": "550e8400-..." }
```

Response `200`: same shape as login. **Old refresh token is revoked immediately.**

### Logout

```
POST /api/auth/logout
{ "refreshToken": "550e8400-..." }
```

Response `204`.

### Current user — `GET /api/auth/me`

```
GET /api/auth/me
Authorization: Bearer <accessToken>
```

Response `200`:
```json
{
  "id":              1,
  "username":        "alice",
  "email":           "alice@caseflow.dev",
  "fullName":        "Alice Admin",
  "roleId":          1,
  "roleCode":        "ADMIN",
  "roleName":        "Admin",
  "permissionCodes": [
    "ADMIN_CONFIG",
    "ADMIN_POOL_VIEW",
    "CUSTOMER_REPLY_SEND",
    "DATA_EXPORT",
    "EMAIL_CONFIG_MANAGE",
    "EMAIL_CONFIG_VIEW",
    "EMAIL_OPERATIONS_MANAGE",
    "EMAIL_OPERATIONS_VIEW",
    "GROUP_MANAGE",
    "INTERNAL_NOTE_ADD",
    "REPORT_VIEW",
    "ROLE_MANAGE",
    "TICKET_ASSIGN",
    "TICKET_CLOSE",
    "TICKET_EMAIL_REPLY_SEND",
    "TICKET_EMAIL_VIEW",
    "TICKET_PRIORITY_CHANGE",
    "TICKET_READ",
    "TICKET_STATUS_CHANGE",
    "TICKET_TRANSFER",
    "USER_MANAGE"
  ],
  "ticketScope": "ALL",
  "groupIds":    [1, 2]
}
```

**Rules:**
- `permissionCodes` is sorted **alphabetically**. Use `includes()` to check membership; do not rely on position.
- **Gate all UI features on `permissionCodes` — never use `roleCode` for access decisions.** `roleCode` and `roleName` are display-only.
- `ticketScope` controls which tickets the user can see. Values: `ALL` | `OWN_GROUPS` | `OWN_AND_OWN_GROUPS` | `ASSIGNED_ONLY`.

---

## Permission Catalog

The backend enforces `PERM_<code>` Spring Security authorities. The FE uses the bare code strings from `permissionCodes`.

| Permission code             | Enforced by                                                        |
|-----------------------------|--------------------------------------------------------------------|
| `TICKET_READ`               | All ticket read endpoints (subject to `ticketScope`)               |
| `TICKET_ASSIGN`             | `POST /tickets/{id}/assign`                                        |
| `TICKET_TRANSFER`           | `POST /tickets/{id}/transfer`                                      |
| `TICKET_STATUS_CHANGE`      | `POST /tickets/{id}/status`, `POST /tickets/{id}/reopen`           |
| `TICKET_CLOSE`              | `POST /tickets/{id}/close`                                         |
| `TICKET_PRIORITY_CHANGE`    | `PUT /tickets/{id}` priority field                                 |
| `INTERNAL_NOTE_ADD`         | `POST /tickets/{id}/notes`                                         |
| `ADMIN_POOL_VIEW`           | Unassigned ticket pool query                                       |
| `EMAIL_CONFIG_VIEW`         | `GET /admin/mailboxes`, `GET /customers/{id}/email-settings`       |
| `EMAIL_CONFIG_MANAGE`       | Write ops on mailboxes and customer email settings/rules           |
| `EMAIL_OPERATIONS_VIEW`     | `GET /admin/ingress-events`                                        |
| `EMAIL_OPERATIONS_MANAGE`   | Replay / quarantine / release ingress events; `POST /emails/ingest` |
| `TICKET_EMAIL_VIEW`         | `GET /tickets/{id}/email/thread` and detail endpoints              |
| `TICKET_EMAIL_REPLY_SEND`   | `POST /tickets/{id}/email/reply`                                   |

### Starter role defaults (seeded by V10 migration)

| Role       | Key permissions                                                                                   |
|------------|---------------------------------------------------------------------------------------------------|
| ADMIN      | All permissions                                                                                   |
| SUPERVISOR | TICKET_READ + workflow + EMAIL_CONFIG_VIEW + EMAIL_OPERATIONS_VIEW + TICKET_EMAIL_*              |
| AGENT      | TICKET_READ + workflow + INTERNAL_NOTE_ADD + TICKET_EMAIL_VIEW + TICKET_EMAIL_REPLY_SEND         |
| VIEWER     | TICKET_READ + TICKET_EMAIL_VIEW                                                                   |

---

## Pagination

`GET /api/tickets` and `GET /api/admin/ingress-events` return paginated responses.

```json
{
  "items":         [ ...items ],
  "page":          0,
  "size":          20,
  "totalElements": 150,
  "totalPages":    8
}
```

Pass `page`, `size`, `sort`, `direction` as query params. Defaults: page=0, size=20.

---

## Error Response

```json
{
  "timestamp": "2026-03-29T10:00:00Z",
  "status":    404,
  "error":     "Not Found",
  "code":      "TICKET_NOT_FOUND",
  "message":   "Ticket not found: 99",
  "path":      "/api/tickets/99",
  "details":   [],
  "requestId": "abc-123"
}
```

Validation errors (400) populate `details`:
```json
"details": [
  { "field": "subject", "message": "must not be blank", "rejectedValue": "", "code": "NotBlank" }
]
```

Every response carries `X-Correlation-Id` header.

### Error codes

| Code                       | Status | Trigger                                          |
|----------------------------|--------|--------------------------------------------------|
| `TICKET_NOT_FOUND`         | 404    | Unknown ticket ID                                |
| `CUSTOMER_NOT_FOUND`       | 404    | Unknown customer ID                              |
| `CONTACT_NOT_FOUND`        | 404    | Unknown contact ID                               |
| `USER_NOT_FOUND`           | 404    | Unknown user ID                                  |
| `GROUP_NOT_FOUND`          | 404    | Unknown group ID                                 |
| `NOTE_NOT_FOUND`           | 404    | Unknown note ID                                  |
| `ATTACHMENT_NOT_FOUND`     | 404    | Unknown attachment ID                            |
| `MAILBOX_NOT_FOUND`        | 404    | Unknown mailbox ID                               |
| `INGRESS_EVENT_NOT_FOUND`  | 404    | Unknown ingress event ID                         |
| `DISPATCH_NOT_FOUND`       | 404    | Unknown dispatch ID                              |
| `ROUTING_RULE_NOT_FOUND`   | 404    | Unknown routing rule ID or customer mismatch     |
| `ASSIGNMENT_CONFLICT`      | 409    | Active assignment already exists                 |
| `DUPLICATE_EMAIL`          | 409    | `messageId` already ingested                     |
| `INVALID_TICKET_STATE`     | 422    | Illegal state transition                         |
| `VALIDATION_FAILED`        | 400    | Bean validation error                            |
| `MALFORMED_REQUEST`        | 400    | Request body missing or unparseable              |
| `MISSING_PARAMETER`        | 400    | Required query parameter absent                  |
| `TYPE_MISMATCH`            | 400    | Path/query parameter type error                  |
| `UNAUTHORIZED`             | 401    | No or invalid Bearer token                       |
| `INVALID_CREDENTIALS`      | 401    | Bad username/password or expired token           |
| `FORBIDDEN`                | 403    | Authenticated but insufficient permission        |
| `EMAIL_DISPATCH_FAILED`    | 503    | SMTP not configured or unreachable               |
| `INTERNAL_ERROR`           | 500    | Unexpected server error                          |

---

## PRIMARY ENDPOINTS — Email Platform

### Ticket Email Thread — PRIMARY FE contract

```
Base: /api/tickets/{ticketId}/email
```

All endpoints require `TICKET_EMAIL_VIEW` permission + `ticketScope` on the ticket.

| Method | Path                   | Permission              | Body / Notes                                                           | Response               |
|--------|------------------------|-------------------------|------------------------------------------------------------------------|------------------------|
| GET    | /thread                | TICKET_EMAIL_VIEW       | Merged inbound+outbound chronological timeline. **Use this.**         | `EmailThreadItem[]`    |
| GET    | /inbound/{eventId}     | TICKET_EMAIL_VIEW       | Full detail for one inbound event                                      | `IngressEventResponse` |
| GET    | /outbound/{dispatchId} | TICKET_EMAIL_VIEW       | Full detail for one outbound dispatch                                  | `DispatchResponse`     |
| POST   | /reply                 | TICKET_EMAIL_REPLY_SEND | Send outbound reply from this ticket                                   | 202 Accepted           |

**Compatibility-only (do not build new UI on these):**

| Method | Path         | Notes                                        |
|--------|--------------|----------------------------------------------|
| GET    | /inbound     | Inbound event list without outbound context  |
| GET    | /dispatches  | Outbound dispatch list without inbound context |

---

#### `EmailThreadItem` shape (GET /thread response items)

```json
{
  "direction":   "INBOUND",
  "id":          42,
  "messageId":   "<abc@mail.example>",
  "fromAddress": "john@acme.com",
  "toAddress":   null,
  "subject":     "Re: Support ticket",
  "status":      "PROCESSED",
  "timestamp":   "2026-03-29T09:00:00Z",
  "bodyPreview": "Hi, I'm having trouble with..."
}
```

| Field        | INBOUND value                                              | OUTBOUND value                                  |
|--------------|------------------------------------------------------------|-------------------------------------------------|
| `direction`  | `"INBOUND"`                                                | `"OUTBOUND"`                                    |
| `status`     | `IngressEventStatus` name                                  | `DispatchStatus` name                           |
| `timestamp`  | `receivedAt`                                               | `sentAt` (falls back to `createdAt`)            |
| `fromAddress`| raw `From:` header value                                   | mailbox address the reply was sent from         |
| `toAddress`  | `null`                                                     | customer email address                          |
| `bodyPreview`| first ~500 chars from MongoDB EmailDocument (null if not yet processed) | first ~500 chars of `textBody` (null if empty) |

Items are sorted **chronologically ascending** by `timestamp`. `bodyPreview` is `null` if the email body is unavailable (e.g., RECEIVED but Stage-2 not yet run). For the full body, fetch the detail endpoint.

---

#### `SendReplyRequest` (POST /reply body)

```json
{
  "mailboxId":           1,
  "toAddress":           "customer@example.com",
  "subject":             "Re: Support ticket",
  "textBody":            "Thank you for contacting us...",
  "htmlBody":            "<p>Thank you...</p>",
  "inReplyToMessageId":  "<original-msg-id@mail.example>"
}
```

| Field                | Required | Notes                                                                   |
|----------------------|----------|-------------------------------------------------------------------------|
| `mailboxId`          | Yes      | ID of the sending mailbox (from `/admin/mailboxes`)                     |
| `toAddress`          | Yes      | Valid email address                                                     |
| `subject`            | Yes      | Reply subject                                                           |
| `textBody`           | No       | Plain text body                                                         |
| `htmlBody`           | No       | HTML body                                                               |
| `inReplyToMessageId` | No       | `messageId` of the customer email being replied to (for threading)      |

Response: `202 Accepted` with no body. The dispatch is queued and sent asynchronously.

---

### Admin — Mailboxes

```
Base: /api/admin/mailboxes
```

| Method | Path             | Permission          | Body / Notes                     | Response              |
|--------|------------------|---------------------|----------------------------------|-----------------------|
| GET    | /                | EMAIL_CONFIG_VIEW   | `?activeOnly=true` for active-only filter | `MailboxResponse[]`   |
| GET    | /{id}            | EMAIL_CONFIG_VIEW   | —                                | `MailboxResponse`     |
| POST   | /                | EMAIL_CONFIG_MANAGE | `MailboxRequest`                 | `MailboxResponse` 201 |
| PUT    | /{id}            | EMAIL_CONFIG_MANAGE | `MailboxRequest`                 | `MailboxResponse`     |
| PATCH  | /{id}/activate   | EMAIL_CONFIG_MANAGE | —                                | `MailboxResponse`     |
| PATCH  | /{id}/deactivate | EMAIL_CONFIG_MANAGE | —                                | `MailboxResponse`     |
| DELETE | /{id}            | EMAIL_CONFIG_MANAGE | —                                | 204                   |

`smtpPassword` is **never returned** — it is write-only (send only when creating/updating).

---

### Admin — Ingress Events

```
Base: /api/admin/ingress-events
```

> List endpoint returns a **paginated** `PagedResponse<IngressEventResponse>`.

| Method | Path             | Permission              | Body / Notes                                                | Response                              |
|--------|------------------|-------------------------|-------------------------------------------------------------|---------------------------------------|
| GET    | /                | EMAIL_OPERATIONS_VIEW   | `?status=&mailboxId=&ticketId=&page=&size=&sort=receivedAt` | `PagedResponse<IngressEventResponse>` |
| GET    | /{id}            | EMAIL_OPERATIONS_VIEW   | —                                                           | `IngressEventResponse`                |
| POST   | /{id}/process    | EMAIL_OPERATIONS_MANAGE | Replay a FAILED event                                       | `IngressEventResponse`                |
| POST   | /{id}/quarantine | EMAIL_OPERATIONS_MANAGE | `{ "reason": "string" }`                                    | `IngressEventResponse`                |
| POST   | /{id}/release    | EMAIL_OPERATIONS_MANAGE | QUARANTINED → RECEIVED for retry                            | `IngressEventResponse`                |

Default sort: `receivedAt DESC`, page size `20`.

---

### Customer Email Settings

```
Base: /api/customers/{customerId}/email-settings
```

| Method | Path            | Permission          | Body / Notes               | Response                        |
|--------|-----------------|---------------------|----------------------------|---------------------------------|
| GET    | /               | EMAIL_CONFIG_VIEW   | —                          | `CustomerEmailSettingsResponse` |
| PUT    | /               | EMAIL_CONFIG_MANAGE | `CustomerEmailSettingsRequest` | `CustomerEmailSettingsResponse` |
| POST   | /rules          | EMAIL_CONFIG_MANAGE | `RoutingRuleRequest`       | `RoutingRuleResponse` 201       |
| PUT    | /rules/{ruleId} | EMAIL_CONFIG_MANAGE | `RoutingRuleRequest`       | `RoutingRuleResponse`           |
| DELETE | /rules/{ruleId} | EMAIL_CONFIG_MANAGE | —                          | 204                             |

---

### Email Ingest — system/webhook only

```
POST /api/emails/ingest
```

> **Not a UI endpoint.** Called by email provider webhooks or operator scripts.
> Requires `EMAIL_OPERATIONS_MANAGE`.

Idempotent on `messageId`. Returns `202 Accepted` with `IngressEventResponse`.
Stage-2 processing (routing, ticket creation) runs asynchronously via the retry scheduler.

**Legacy email document endpoints (do not use for new UI):**

| Method | Path                     | Permission              | Notes                                                        |
|--------|--------------------------|-------------------------|--------------------------------------------------------------|
| GET    | /emails/{id}             | TICKET_READ             | MongoDB document — returns **sanitized** `sanitizedHtmlBody`; safe to render |
| GET    | /emails/{id}/raw         | EMAIL_OPERATIONS_MANAGE | Raw unsanitized `htmlBody` — **audit/debug only, never render directly** |
| GET    | /emails/by-ticket/{id}   | TICKET_READ             | Legacy list view; use `/tickets/{id}/email/thread` instead   |
| GET    | /emails/by-thread/{key}  | TICKET_READ             | MongoDB thread lookup by threadKey                           |

---

## PRIMARY NON-EMAIL ENDPOINTS

### Auth (`/api/auth`) — no auth required for login/refresh/logout

| Method | Path     | Body                     | Response        |
|--------|----------|--------------------------|-----------------|
| POST   | /login   | `{username, password}`   | `TokenResponse` |
| POST   | /refresh | `{refreshToken}`         | `TokenResponse` |
| POST   | /logout  | `{refreshToken}`         | 204             |
| GET    | /me      | — (Bearer required)      | `MeResponse`    |

### Tickets (`/api/tickets`) — `TICKET_READ` + ticketScope

| Method | Path                     | Permission           | Body / Notes                                    | Response                               |
|--------|--------------------------|----------------------|-------------------------------------------------|----------------------------------------|
| POST   | /                        | TICKET_READ          | `{subject, description?, priority, customerId}` | `TicketResponse` 201                   |
| GET    | /                        | TICKET_READ          | query params (see Pagination)                   | `PagedResponse<TicketSummaryResponse>` |
| GET    | /{id}                    | TICKET_READ          | —                                               | `TicketResponse`                       |
| GET    | /{id}/detail             | TICKET_READ          | —                                               | `TicketDetailResponse`                 |
| GET    | /by-ticket-no/{ticketNo} | TICKET_READ          | —                                               | `TicketResponse`                       |
| PUT    | /{id}                    | TICKET_READ          | `{subject?, description?, priority?}`           | `TicketResponse`                       |
| POST   | /{id}/status             | TICKET_STATUS_CHANGE | `{status}`                                      | `TicketResponse`                       |
| POST   | /{id}/close              | TICKET_CLOSE         | —                                               | `TicketResponse`                       |
| POST   | /{id}/reopen             | TICKET_STATUS_CHANGE | —                                               | `TicketResponse`                       |

### Other resources — see `docs/api-endpoints.md` for full shapes

- `GET/POST/PUT/PATCH /api/customers` — `TICKET_READ`
- `GET/POST/PUT/PATCH /api/contacts` — `TICKET_READ`
- `GET/POST/PUT/PATCH /api/users` — `USER_MANAGE`
- `GET/POST/PUT/PATCH /api/groups` — `GROUP_MANAGE`
- `POST /api/tickets/{id}/notes` — `INTERNAL_NOTE_ADD`
- `POST /api/tickets/{id}/assign` — `TICKET_ASSIGN`
- `POST /api/tickets/{id}/transfer` — `TICKET_TRANSFER`
- `POST/GET/DELETE /api/attachments` — `TICKET_READ`

---

## Key Response Shapes

### `MeResponse`
```json
{
  "id": 1, "username": "alice", "email": "alice@caseflow.dev",
  "fullName": "Alice Admin", "roleId": 1, "roleCode": "ADMIN", "roleName": "Admin",
  "permissionCodes": ["EMAIL_CONFIG_VIEW", "TICKET_READ", "..."],
  "ticketScope": "ALL",
  "groupIds": [1, 2]
}
```

### `IngressEventResponse`
```json
{
  "id": 1, "mailboxId": 2,
  "messageId": "<abc@mail.example>", "rawFrom": "john@acme.com",
  "rawSubject": "Help with order", "receivedAt": "2026-03-29T09:00:00Z",
  "status": "PROCESSED", "failureReason": null,
  "processingAttempts": 1, "lastAttemptAt": "2026-03-29T09:00:05Z",
  "processedAt": "2026-03-29T09:00:05Z",
  "documentId": "mongo-object-id", "ticketId": 42
}
```

### `DispatchResponse`
```json
{
  "id": 10, "ticketId": 42,
  "messageId": "<uuid@caseflow>",
  "fromAddress": "support@caseflow.dev", "toAddress": "john@acme.com",
  "subject": "Re: Help with order", "status": "SENT",
  "attempts": 1, "lastAttemptAt": "2026-03-29T10:00:01Z",
  "sentAt": "2026-03-29T10:00:01Z", "failureReason": null,
  "scheduledAt": "2026-03-29T10:00:00Z", "createdAt": "2026-03-29T10:00:00Z"
}
```

### `MailboxResponse`
```json
{
  "id": 1, "name": "Support Inbox", "displayName": "CaseFlow Support",
  "address": "support@caseflow.dev",
  "providerType": "SMTP_RELAY", "inboundMode": "WEBHOOK", "outboundMode": "SMTP",
  "isActive": true, "defaultGroupId": 3, "defaultPriority": "MEDIUM",
  "smtpHost": "smtp.example.com", "smtpPort": 587, "smtpUsername": "user",
  "smtpUseSsl": false,
  "lastSuccessfulInboundAt": "2026-03-29T08:00:00Z",
  "lastSuccessfulOutboundAt": "2026-03-29T09:30:00Z",
  "createdAt": "2026-03-01T00:00:00Z", "updatedAt": "2026-03-29T08:00:00Z"
}
```

`smtpPassword` is **never** in the response.

### `CustomerEmailSettingsResponse`
```json
{
  "id": 1, "customerId": 5,
  "unknownSenderPolicy": "MANUAL_REVIEW", "matchingStrategy": "CONTACT_FIRST",
  "isActive": true, "trustedContactsOnly": false,
  "autoCreateContact": false, "allowSubdomains": false,
  "defaultGroupId": 3, "defaultPriority": "MEDIUM",
  "updatedAt": "2026-03-29T08:00:00Z",
  "rules": [
    {
      "id": 1, "customerId": 5, "senderMatchType": "EXACT_EMAIL",
      "matchValue": "support@bigcorp.com", "priority": 10,
      "isActive": true, "createdAt": "2026-03-01T00:00:00Z", "updatedAt": null
    }
  ]
}
```

### `RoutingRuleRequest` (POST/PUT /rules body)
```json
{
  "senderMatchType": "EXACT_EMAIL",
  "matchValue":      "support@bigcorp.com",
  "priority":        10,
  "isActive":        true
}
```

### `MailboxRequest` (POST/PUT /mailboxes body)
```json
{
  "name":         "Support Inbox",
  "displayName":  "CaseFlow Support",
  "address":      "support@caseflow.dev",
  "providerType": "SMTP_RELAY",
  "inboundMode":  "WEBHOOK",
  "outboundMode": "SMTP",
  "isActive":     true,
  "defaultGroupId":  3,
  "defaultPriority": "MEDIUM",
  "smtpHost":     "smtp.example.com",
  "smtpPort":     587,
  "smtpUsername": "user",
  "smtpPassword": "secret",
  "smtpUseSsl":   false
}
```

`smtpPassword` is only sent when creating or updating a mailbox. Omit (or send `null`) to keep the existing password on updates.

---

## Enums

```
TicketStatus        : NEW | TRIAGED | ASSIGNED | IN_PROGRESS | WAITING_CUSTOMER | RESOLVED | CLOSED | REOPENED
TicketPriority      : LOW | MEDIUM | HIGH | CRITICAL
TicketScope         : ALL | OWN_GROUPS | OWN_AND_OWN_GROUPS | ASSIGNED_ONLY
NoteType            : INFO | INVESTIGATION | ESCALATION | INTERNAL
GroupType           : TRADE | OPERATIONS | SUPPORT
IngressEventStatus  : RECEIVED | PROCESSING | PROCESSED | FAILED | QUARANTINED
DispatchStatus      : PENDING | SENDING | SENT | FAILED | PERMANENTLY_FAILED
ProviderType        : SMTP_RELAY | SENDGRID | MAILGUN
InboundMode         : WEBHOOK | IMAP_POLL | MANUAL
OutboundMode        : SMTP | API
SenderMatchType     : EXACT_EMAIL | DOMAIN
MatchingStrategy    : CONTACT_FIRST | RULE_FIRST
UnknownSenderPolicy : MANUAL_REVIEW | IGNORE | REJECT
```

All enums serialize as their string name (Spring Boot default).

---

## CORS

Pre-configured: `http://localhost:3000`, `http://localhost:5173`.

---

## Dev Seed Data

Start with `SPRING_PROFILES_ACTIVE=dev`:

| Entity    | Count | Details                                                                                |
|-----------|-------|----------------------------------------------------------------------------------------|
| Groups    | 2     | Support (id=1), Operations (id=2)                                                      |
| Users     | 3     | alice/admin123 (ADMIN), bob/agent123 (AGENT), carol/viewer123 (VIEWER)                |
| Customers | 2     | ACME Corp, Globex Inc                                                                  |
| Contacts  | 3     | john.doe@acme.com, jane.doe@acme.com, burns@globex.com                                |
| Tickets   | 3     | TKT-0001 (NEW/CRITICAL), TKT-0002 (TRIAGED/HIGH), TKT-0003 (IN_PROGRESS/MEDIUM)      |
| Notes     | 2     | One on TKT-0001, one on TKT-0003                                                       |
| Emails    | 1     | Linked to TKT-0001                                                                     |

> Note: Seed data hard-codes ticket numbers. Production tickets use the sequence format `TKT-0000001`.

---

## What NOT to send in request bodies

- `createdBy`, `performedBy`, `assignedBy`, `transferredBy` — resolved from JWT
- `ticketNo` on create — server-managed (`TKT-{7-digit-sequence}`)
- `status` on create — always starts as `NEW`
- `smtpPassword` in GET responses — write-only

---

## Legacy / Compat endpoints — do not build new UI on these

| Endpoint                         | Status  | Replace with                                      |
|----------------------------------|---------|---------------------------------------------------|
| GET `/tickets/{id}/email/inbound`   | compat  | GET `/tickets/{id}/email/thread`                |
| GET `/tickets/{id}/email/dispatches`| compat  | GET `/tickets/{id}/email/thread`                |
| GET `/emails/by-ticket/{id}`     | legacy  | GET `/tickets/{id}/email/thread`                  |
| GET `/emails/by-thread/{key}`    | legacy  | Internal/ops use only; no FE equivalent needed    |
| GET `/emails/{id}`               | legacy  | Returns sanitized body (`sanitizedHtmlBody`) — safe to render  |
| GET `/emails/{id}/raw`           | debug   | Raw unsanitized HTML — audit only, never render directly        |
