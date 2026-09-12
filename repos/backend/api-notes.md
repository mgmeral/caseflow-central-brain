# CaseFlow API Notes

## Base URL

```
http://localhost:8080/api
```

## Interactive Documentation

Swagger UI: `http://localhost:8080/swagger-ui.html`
OpenAPI JSON: `http://localhost:8080/v3/api-docs`

---

## Authentication

CaseFlow uses **JWT Bearer authentication**.

1. Call `POST /api/auth/login` with username + password → receive `accessToken` and `refreshToken`.
2. Include `Authorization: Bearer <accessToken>` on every subsequent request.
3. When the access token expires (1 hour), call `POST /api/auth/refresh` with the refresh token to receive a rotated token pair.
4. Call `POST /api/auth/logout` to revoke the refresh token on sign-out.

### Dev credentials (dev profile)

| Username | Password   | Role   | Permissions                                   |
|----------|------------|--------|-----------------------------------------------|
| alice    | admin123   | ADMIN  | Full access including DELETE                  |
| bob      | agent123   | AGENT  | Read + operational mutations (POST/PUT/PATCH) |
| carol    | viewer123  | VIEWER | Read-only (GET)                               |

### Role permission map

| Role   | Allowed methods            |
|--------|----------------------------|
| ADMIN  | GET · POST · PUT · PATCH · DELETE |
| AGENT  | GET · POST · PUT · PATCH   |
| VIEWER | GET only                   |

---

## Error Response Shape

All errors return this consistent JSON structure:

```json
{
  "timestamp": "2024-06-01T12:00:00Z",
  "status": 404,
  "error": "Not Found",
  "code": "TICKET_NOT_FOUND",
  "message": "Ticket not found: 42",
  "path": "/api/tickets/42",
  "details": null,
  "requestId": "a1b2c3d4-..."
}
```

### Validation Error (400)

```json
{
  "timestamp": "...",
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDATION_FAILED",
  "message": "Request validation failed",
  "path": "/api/tickets",
  "details": [
    {
      "field": "subject",
      "message": "must not be blank",
      "rejectedValue": "",
      "code": "NotBlank.createTicketRequest.subject"
    }
  ],
  "requestId": "..."
}
```

### Error Codes Reference

| Code                            | Status | Description                           |
|---------------------------------|--------|---------------------------------------|
| TICKET_NOT_FOUND                | 404    | Ticket does not exist                 |
| CUSTOMER_NOT_FOUND              | 404    | Customer does not exist               |
| CONTACT_NOT_FOUND               | 404    | Contact does not exist                |
| USER_NOT_FOUND                  | 404    | User does not exist                   |
| GROUP_NOT_FOUND                 | 404    | Group does not exist                  |
| NOTE_NOT_FOUND                  | 404    | Note does not exist                   |
| ATTACHMENT_NOT_FOUND            | 404    | Attachment does not exist             |
| ASSIGNMENT_CONFLICT             | 409    | Active assignment already exists      |
| INVALID_TICKET_STATE            | 422    | State transition not allowed          |
| VALIDATION_FAILED               | 400    | Bean validation errors                |
| MALFORMED_REQUEST               | 400    | Request body missing or malformed     |
| MISSING_PARAMETER               | 400    | Required query parameter missing      |
| TYPE_MISMATCH                   | 400    | Path/query parameter type mismatch    |
| UNAUTHORIZED                    | 401    | Authentication required               |
| FORBIDDEN                       | 403    | Insufficient permissions              |
| INTERNAL_ERROR                  | 500    | Unexpected server error               |

---

## Endpoints Overview

### Auth — `/api/auth`

| Method | Path                  | Role        | Description                                |
|--------|-----------------------|-------------|--------------------------------------------|
| POST   | /api/auth/login       | Public      | Authenticate; returns token pair           |
| POST   | /api/auth/refresh     | Public      | Rotate token pair using refresh token      |
| POST   | /api/auth/logout      | Public      | Revoke refresh token                       |
| GET    | /api/auth/me          | Any auth    | Return current user info                   |

### Tickets — `/api/tickets`

| Method | Path                          | Role   | Description                       |
|--------|-------------------------------|--------|-----------------------------------|
| GET    | /api/tickets                  | VIEWER | List all tickets (filterable)     |
| GET    | /api/tickets/{id}             | VIEWER | Get ticket by ID                  |
| GET    | /api/tickets/by-ticket-no/{no}| VIEWER | Get ticket by ticket number       |
| POST   | /api/tickets                  | AGENT  | Create a new ticket               |
| PUT    | /api/tickets/{id}             | AGENT  | Update ticket subject/description |
| POST   | /api/tickets/{id}/status      | AGENT  | Change ticket status              |
| POST   | /api/tickets/{id}/close       | AGENT  | Close a ticket                    |
| POST   | /api/tickets/{id}/reopen      | AGENT  | Reopen a closed ticket            |

**Filter params for GET /api/tickets:** `status`, `priority`, `userId`, `groupId`, `customerId`, `search`, `from`, `to`, `page` (default 0), `size` (default 20, max 100), `sort` (default `createdAt`), `direction` (`asc`/`desc`, default `desc`)

### Customers — `/api/customers`

| Method | Path                        | Role   | Description              |
|--------|-----------------------------|--------|--------------------------|
| GET    | /api/customers              | VIEWER | List all customers       |
| GET    | /api/customers/{id}         | VIEWER | Get customer by ID       |
| POST   | /api/customers              | AGENT  | Create customer          |
| PUT    | /api/customers/{id}         | AGENT  | Update customer          |
| PATCH  | /api/customers/{id}/activate| AGENT  | Activate customer        |
| PATCH  | /api/customers/{id}/deactivate| AGENT| Deactivate customer      |

### Contacts — `/api/contacts`

| Method | Path                        | Role   | Description              |
|--------|-----------------------------|--------|--------------------------|
| GET    | /api/contacts               | VIEWER | List all contacts        |
| GET    | /api/contacts/{id}          | VIEWER | Get contact by ID        |
| GET    | /api/contacts/by-email      | VIEWER | Find contact by email    |
| GET    | /api/contacts/by-customer/{id}| VIEWER | Contacts for customer  |
| POST   | /api/contacts               | AGENT  | Create contact           |
| PUT    | /api/contacts/{id}          | AGENT  | Update contact           |

### Users — `/api/users`

| Method | Path                        | Role  | Description           |
|--------|-----------------------------|-------|-----------------------|
| GET    | /api/users                  | VIEWER| List all users        |
| GET    | /api/users/{id}             | VIEWER| Get user by ID        |
| GET    | /api/users/by-username      | VIEWER| Find by username      |
| GET    | /api/users/by-email         | VIEWER| Find by email         |
| POST   | /api/users                  | ADMIN | Create user           |
| PUT    | /api/users/{id}             | ADMIN | Update user           |
| PATCH  | /api/users/{id}/activate    | ADMIN | Activate user         |
| PATCH  | /api/users/{id}/deactivate  | ADMIN | Deactivate user       |

### Groups — `/api/groups`

| Method | Path                           | Role  | Description        |
|--------|--------------------------------|-------|--------------------|
| GET    | /api/groups                    | VIEWER| List active groups |
| GET    | /api/groups/{id}               | VIEWER| Get group by ID    |
| GET    | /api/groups/by-type            | VIEWER| Filter by type     |
| POST   | /api/groups                    | ADMIN | Create group       |
| PUT    | /api/groups/{id}               | ADMIN | Update group       |
| PATCH  | /api/groups/{id}/activate      | ADMIN | Activate group     |
| PATCH  | /api/groups/{id}/deactivate    | ADMIN | Deactivate group   |

### Notes — `/api/notes`

| Method | Path                          | Role  | Description             |
|--------|-------------------------------|-------|-------------------------|
| GET    | /api/notes/{id}               | VIEWER| Get note by ID          |
| GET    | /api/notes/by-ticket/{id}     | VIEWER| List notes for ticket   |
| POST   | /api/notes                    | AGENT | Add note to ticket      |

### Assignments — `/api/assignments`

| Method | Path                               | Role  | Description               |
|--------|------------------------------------|-------|---------------------------|
| GET    | /api/assignments/by-ticket/{id}    | VIEWER| Get active assignment     |
| POST   | /api/assignments/assign            | AGENT | Assign ticket             |
| POST   | /api/assignments/reassign          | AGENT | Reassign ticket           |
| POST   | /api/assignments/unassign          | AGENT | Unassign ticket           |

### Transfers — `/api/transfers`

| Method | Path                         | Role  | Description                |
|--------|------------------------------|-------|----------------------------|
| GET    | /api/transfers/by-ticket/{id}| VIEWER| Get transfer history       |
| POST   | /api/transfers               | AGENT | Transfer ticket to group   |

### Emails — `/api/emails`

| Method | Path                         | Role  | Description                                     |
|--------|------------------------------|-------|-------------------------------------------------|
| POST   | /api/emails/ingest           | AGENT | Ingest an email (idempotent on messageId → 409) |
| GET    | /api/emails/{id}             | VIEWER| Get email document by ID                        |
| GET    | /api/emails/by-ticket/{id}   | VIEWER| Get emails linked to ticket                     |
| GET    | /api/emails/by-thread/{key}  | VIEWER| Get emails by thread key                        |

### Attachments — `/api/attachments`

| Method | Path                              | Role  | Description                              |
|--------|-----------------------------------|-------|------------------------------------------|
| POST   | /api/attachments/upload           | AGENT | Upload file (multipart, max 25 MB)       |
| GET    | /api/attachments/{id}             | VIEWER| Get attachment metadata                  |
| GET    | /api/attachments/by-ticket/{id}   | VIEWER| List attachments for a ticket            |
| GET    | /api/attachments/{id}/download    | VIEWER| Download attachment binary               |
| DELETE | /api/attachments/{id}             | ADMIN | Delete attachment                        |

---

## CORS

Allowed origins (configurable via `caseflow.cors.allowed-origins`):
- `http://localhost:3000` (Create React App default)
- `http://localhost:5173` (Vite default)

---

## Enum Values

| Enum           | Values                                                                         |
|----------------|--------------------------------------------------------------------------------|
| TicketStatus   | NEW, TRIAGED, ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED, REOPENED |
| TicketPriority | LOW, MEDIUM, HIGH, CRITICAL                                                    |
| NoteType       | INFO, INVESTIGATION, ESCALATION, INTERNAL                                      |
| GroupType      | TRADE, OPERATIONS, SUPPORT                                                     |

---

## Seed Data (dev profile)

Run with `--spring.profiles.active=dev` to load seed data automatically on startup.

Seeded: 3 groups · 3 users · 2 customers · 3 contacts · 3 tickets · 2 notes · 1 sample email

---

## Storage

Attachments are stored locally under `./storage-data` by default (MinIO in Docker Compose).
Configure the storage root with `caseflow.storage.root-path`.

Upload/download flow:
- Upload: `POST /api/attachments/upload` — multipart `file` field, max 25 MB, returns `AttachmentMetadataResponse`
- Download: `GET /api/attachments/{id}/download` — streams binary with original `Content-Type`
- Object key pattern: `tickets/{ticketId}/{uuid}_{sanitizedName}`
