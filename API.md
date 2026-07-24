# tybird CRM App — API Specification V1

**Owner:** billiton internet services GmbH
**Repository path:** `billiton-gmbh/tybird-api/API.md` (this file — the single source of truth)
**Last updated:** 2026-07-24
**Status:** Version 1 — deliberately small, extended incrementally

> **What changed in this update (2026-06-23):** The platform host was consolidated. The
> single live entry point is now **`https://api.tybird.com/api/v1/`** — note the **`/api/v1/`**
> path, **not** `/v1/`. The previous `meinbbh...` host is **no longer the API base** and must
> not be used for the REST API. Authentication, endpoints and JSON shapes are **unchanged** —
> you keep building against the same contract. A short **Quick test** block was added in §2 so
> you can smoke-test auth plus a first call in a minute (with a Windows PowerShell note).

---

## 1. Overview

This REST API is the single integration surface for the CRM. Clients (a native staff app,
external integrations) never talk to the database directly — only to this API. billiton
builds and operates the API layer.

The API exposes a **curated, stable view** of the data. Internal/system fields (lead
scoring, duplicate detection, spam flags, UTM tracking, internal timestamps) are
intentionally not exposed. Clients always work against a clean contract.

### The product context: tybird (multi-tenant)

The CRM is a multi-tenant SaaS platform under the name **tybird**. Büdenbender Hausbau (BBH)
is the first tenant; more tenants follow. For a client this means:

- Every account belongs to one or more **tenants**. The active tenant is derived from the
  caller's credentials (see §2 "Tenant scope") — the client never sends a tenant id.
- All data the API returns is **scoped to the active tenant**. A caller for tenant A can
  only ever see tenant A's data.
- A single app codebase serves all tenants; the tenant model is the foundation for
  onboarding further tenants without a separate app.

### Scope of V1: Contacts + Todos + Card

The first scope is three core areas: **Contacts**, **Todos (tasks)**, and **Card** (a
field-facing contact hub per sales rep). This spec currently covers Contacts and Tasks
(Phase 1). The Card endpoints are described as a concept in §6 so an app can be architected
around three areas, not two.

### Goals, in order

1. **Phase 1 (live):** View contacts and the tasks of a contact; a user's own task list;
   update a task (mark done, reschedule).
2. **Phase 2 (next):** Push notifications (device registration + delivery).
3. **Later (concept):** Card endpoints; end-customer access path (see §6).

Phase 1 is fully specified below and is ready to build against. Phase 2 and later are
described as concepts so the app architecture can plan for them; they are finalized in
scoped later versions.

---

## 2. Conventions (apply to all endpoints)

| Topic              | Definition |
|--------------------|------------|
| Base URL           | `https://api.tybird.com/api/v1/` — the single live product entry point. Note the `/api/v1/` path. |
| Format             | JSON for request and response, UTF-8 |
| Authentication     | Two models — **user login (JWT)** or **API key** — both via the `Authorization` header (see below) |
| Authorization      | Two scopes — **staff** (tenant-wide) and **end-customer (self)** (own records only). See "Authorization scopes" below. |
| Tenant scope       | Derived from the credentials (active tenant). Enforced on every read and write. |
| IDs                | UUID (string) |
| Date/time          | ISO-8601 in UTC, e.g. `2026-06-14T09:30:00Z` |
| Pagination         | Query params `page` (from 1) and `limit` (default 25, max 100) |
| Error shape        | `{ "error": { "code": "string", "message": "string" } }` |

> **One host.** `https://api.tybird.com/api/v1/` is the single entry point. A thin gateway
> sits in front of the backend and also passes through the authentication endpoint, so login
> works at `https://api.tybird.com/auth/v1/token`. The earlier `meinbbh...` host is being
> retired for the API — do not target it. A request without valid credentials returns the
> standard `401` error shape; a request to the wrong path (e.g. `/v1/...` instead of
> `/api/v1/...`) returns `404`.

### Quick test (smoke test)

Two calls confirm your setup: get a token, then read contacts.

**curl (macOS/Linux — or `curl.exe` on Windows):**

```bash
# 1) get an access token (user login)
curl -s -X POST "https://api.tybird.com/auth/v1/token?grant_type=password" \
  -H "Content-Type: application/json" \
  -d '{"email":"YOU@example.com","password":"YOUR_PASSWORD"}'

# 2) read contacts with the access_token from step 1
curl -s "https://api.tybird.com/api/v1/contacts" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**Windows PowerShell note.** There, `curl` is an alias for `Invoke-WebRequest` and does
**not** accept `-H`/`-d`; quoting JSON inline also breaks the body. Use `curl.exe` (the real
curl ships with Windows) or, simpler, `Invoke-RestMethod`:

```powershell
$body = @{ email = "YOU@example.com"; password = "YOUR_PASSWORD" } | ConvertTo-Json
$r = Invoke-RestMethod -Method Post -Uri "https://api.tybird.com/auth/v1/token?grant_type=password" -ContentType "application/json" -Body $body
Invoke-RestMethod -Uri "https://api.tybird.com/api/v1/contacts" -Headers @{ Authorization = "Bearer $($r.access_token)" }
```

A successful step 2 returns the tenant's contacts (the `data` array). `401` with the error
shape means the token is missing or invalid; `404` means the path is wrong — check it is
`/api/v1/...`, not `/v1/...`.

### Authentication — two models

The API accepts two kinds of credentials in the same header:

```
Authorization: Bearer <credential>
```

The backend distinguishes them by the credential itself: a value starting with `tyb_live_`
is treated as an API key; anything else is validated as a Supabase JWT.

#### Model 1 — User login (JWT)

For clients with **human users** (e.g. a staff app where each employee signs in). The user
authenticates via Supabase Auth (email + password) at `https://api.tybird.com/auth/v1/token`
and the app sends the resulting access token (JWT) with every request. Properties:

- **Identity:** the request is bound to a specific user (`user_id` derived from the token).
- **Reads can be user-scoped:** the `mine` parameter and the "own task list" endpoint
  return only the calling user's records.
- **Writes are allowed** (`PATCH /tasks/{id}`), and are restricted to the user's own
  records.
- **Token lifetime:** access tokens are short-lived and must be refreshed via the Supabase
  refresh-token flow (the Supabase client SDKs handle this). The Supabase project URL and
  public anon key (for the login flow) are provided separately.
- **Tenant:** carried in the token as the `active_tenant_id` claim, set by the backend at
  login. The client never sends a tenant id and cannot select a foreign tenant.

#### Model 2 — API key

For **server-to-server integrations** (no human user; e.g. a sync job, a middleware, an
automation). A tenant admin creates a long-lived, tenant-scoped key in the CRM
(Settings → API-Keys); the key is shown once at creation and stored only as a hash.
Properties:

- **No user identity:** the key is bound to a **tenant**, not a person.
- **Reads are tenant-wide:** there is no user to scope to, so `mine` has no effect and the
  "own task list" endpoint returns **all** tasks of the tenant.
- **Writes are rejected:** any write (`PATCH /tasks/{id}`) returns `403 forbidden`
  ("write access requires a user token").
- **Tenant:** derived from the key. The key never sees another tenant's data.
- **Transport:** send the key as `Authorization: Bearer tyb_live_...`. Treat it like a
  password; it grants tenant-wide read access until revoked.

> **Which model to use.** Use **Model 1** for any app whose users are people who view and
> edit their own data — it provides per-user identity and write access. Use **Model 2** only
> for unattended system-to-system integrations that read tenant data and do not need to
> write through the API.

### Tenant scope

Every account belongs to an active **tenant**. With a user login the tenant comes from the
token's `active_tenant_id` claim; with an API key it comes from the key. The API derives the
tenant from the credential — **the client never sends a tenant id** — and constrains every
read and write to it. A request whose credential carries no resolvable tenant is rejected
(`403 forbidden`, fail-closed). This is enforced now (not a future step).

### Authorization scopes

The API distinguishes two authorization scopes, both under the user-login (JWT) model:

- **Staff scope** — internal staff accounts (Hausberater, Innendienst, admins). Reads are
  tenant-wide (or user-scoped via `mine`); writes are allowed per the endpoint. This is the
  scope the Contacts and Tasks endpoints are built for.
- **End-customer (self) scope** — a person who owns their own records (a customer with an app
  login). Every read and write is constrained to **their own** data: their own contact, their
  own documents, their own referrals, their own requests. A self-scope caller can never see
  another person's data or the tenant's wider data.

The scope is derived from the account, never sent by the client. Endpoints note per-scope
behaviour inline where it differs. The API-key model (Model 2) is always staff/tenant scope.

### Standard error codes

| HTTP | code               | Meaning |
|------|--------------------|---------|
| 400  | `validation_error` | Invalid input (details in `message`) |
| 401  | `unauthorized`     | Credential missing or invalid |
| 403  | `forbidden`        | Authenticated but not allowed — e.g. not a staff account, no active tenant, an API key attempting a write, or editing a record that is not the user's own |
| 404  | `not_found`        | Resource does not exist (or belongs to another tenant — existence is not revealed), or the request path is wrong (use `/api/v1/...`) |
| 409  | `conflict`         | e.g. duplicate record |
| 500  | `server_error`     | Internal error |

---

## 3. Data model notes

A **contact** (`contact`) is the central entity (a lead / prospect / customer). The personal
data (name, email, phone) lives on one or more **contact persons** (`contact_person`); one of
them is the primary person. The API returns the primary person inline within the contact
object, so a second request for the basics is not needed.

A **task** (`task`) may be linked to a contact (`contact_id`) or stand alone (e.g. an
internal task). A task has an assignee, a type, a priority, a status and a due date.

All data is scoped to the active tenant (see §2).

---

## 4. PHASE 1 — Endpoints

> **Per-model behaviour.** Where an endpoint behaves differently under a user login vs. an
> API key, it is noted inline. In short: user login = identity + user-scoped reads + writes;
> API key = tenant-wide reads, no writes.

### 4.0 `GET /me` — active tenant + identity

Loads the **active tenant** (for app branding) and, on a user login, the caller's **own
identity**. Call this right after login to skin the app (accent colour, font, logo) and to
show who is signed in. Read-only.

**Auth models:**

- **User login** — `user` is populated with the caller's profile.
- **API key** — there is no user, so `user` is `null`. The `tenant` block is still returned.

**Response `200`:**

```json
{
  "tenant": {
    "id": "uuid",
    "slug": "bbh",
    "name": "Büdenbender Hausbau",
    "brand": { "accent": "#6e4523", "font": null, "logo_url": null },
    "enabled_modules": ["marketing", "workflows"]
  },
  "user": {
    "id": "uuid",
    "vorname": "...",
    "nachname": "...",
    "email": "...",
    "role": "..."
  }
}
```

With an API key, `user` is `null`.

`brand` is intentionally limited to three public tokens — `accent`, `font`, `logo_url`
(each a string or `null`). Any other tokens stored on the tenant are not exposed here;
further tokens may be added later, additively.

The active tenant is derived from the credentials (see §2 "Tenant scope") — the client never
sends a tenant id, and the response is always scoped to that one tenant.

**End-customer (self) scope:** `/me` additionally returns the caller's assigned **advisor** —
the linked Hausberater, for the app's contact block — derived from the caller's contact
(`hausberater_user_id`):

```json
{
  "advisor": {
    "id": "uuid",
    "vorname": "Thomas",
    "nachname": "Klein",
    "role": "hausberater",
    "avatar_url": "https://..."
  }
}
```

`advisor` is `null` if none is assigned. Staff callers do not receive `advisor`.

### 4.1 Contacts

#### `GET /contacts` — list contacts

Returns a paginated list of contacts, each with its primary person.

**Query parameters (all optional):**

| Parameter     | Type          | Description |
|---------------|---------------|-------------|
| `page`        | int           | Page, from 1 (default 1) |
| `limit`       | int           | Results per page (default 25, max 100) |
| `search`      | string        | Full-text search over name, email, city |
| `status`      | string        | Filter by contact status (see allowed values below) |
| `assigned_to` | string (UUID) | Only contacts of a specific Hausberater |
| `mine`        | bool          | `true` = only contacts whose Hausberater is the logged-in user. **User login only** — with an API key there is no user, so `mine` has no effect (reads stay tenant-wide). |

**Allowed values for `status`** (verified against the database):
`neu`, `in_qualifizierung`, `qualifiziert`, `pool`, `gewonnen`, `verloren`, `tot`,
`abgesagt`, `im_bau`, `uebergeben`.

**Allowed values for `pipeline_stage`** (read-only in V1, verified against the database):
`neuer_kontakt`, `erster_kontakt`, `bedarfsanalyse`, `angebotsvorbereitung`,
`angebot_praesentiert`, `entscheidung_ausstehend`, `gewonnen`, `verloren`, `im_bau`,
`uebergeben`.

**Response (200):**

```json
{
  "data": [
    {
      "id": "uuid",
      "status": "neu",
      "pipeline_stage": "neuer_kontakt",
      "ort": "Siegen",
      "plz": "57072",
      "region": "Siegerland",
      "source": "webform-beratung",
      "hausberater_user_id": "uuid|null",
      "created_at": "2026-06-01T08:00:00Z",
      "primary_person": {
        "id": "uuid",
        "anrede": "Herr",
        "vorname": "Max",
        "nachname": "Mustermann",
        "email": "max@example.de",
        "phone_mobile": "+49...",
        "phone_landline": "+49...|null"
      }
    }
  ],
  "pagination": { "page": 1, "limit": 25, "total": 123 }
}
```

#### `GET /contacts/{id}` — contact detail

Returns a single contact with all display-relevant fields, the primary person, and the full
list of contact persons. A contact that belongs to another tenant returns `404` (its
existence is not revealed).

**Response (200):**

```json
{
  "id": "uuid",
  "status": "neu",
  "pipeline_stage": "neuer_kontakt",
  "ort": "Siegen",
  "plz": "57072",
  "region": "Siegerland",
  "addr_street": "Musterweg",
  "addr_house_no": "12",
  "source": "webform-beratung",
  "notes": "Free-text note from the consultant",
  "hausberater_user_id": "uuid|null",
  "katalog_digital": true,
  "katalog_print": false,
  "created_at": "2026-06-01T08:00:00Z",
  "updated_at": "2026-06-10T11:00:00Z",
  "primary_person": { "...": "same shape as in the list" },
  "persons": [
    { "id": "uuid", "is_primary": true,  "vorname": "Max",   "nachname": "Mustermann", "email": "...", "phone_mobile": "..." },
    { "id": "uuid", "is_primary": false, "vorname": "Erika", "nachname": "Mustermann", "email": "...", "phone_mobile": "..." }
  ]
}
```

> Fields such as scoring, duplicate status, spam flag, UTM tracking and internal timestamps
> are intentionally **not** exposed. If a client needs one of them, ask us — we decide per
> field.

---

### 4.2 Tasks

Tasks are the core of Phase 1. There are two views: the tasks **of a contact**, and a user's
**own task list**.

**Fixed value lists (verified against the database — use exactly these):**

| Field       | Allowed values |
|-------------|----------------|
| `status`    | `open`, `in_progress`, `done`, `cancelled` |
| `priority`  | `low`, `normal`, `high`, `urgent` |
| `task_type` | `call_back`, `send_document`, `schedule_meeting`, `follow_up`, `review`, `approval`, `custom` |

#### `GET /contacts/{id}/tasks` — tasks of a contact

**Query parameters (optional):** `status` (repeatable, e.g. `?status=open&status=in_progress`,
or comma-separated `?status=open,in_progress`)

**Response (200):**

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Call back",
      "description": "Customer asked for a callback in the afternoon",
      "task_type": "call_back",
      "status": "open",
      "priority": "high",
      "due_at": "2026-06-15T14:00:00Z",
      "assignee_id": "uuid|null",
      "contact_id": "uuid",
      "created_at": "2026-06-14T08:00:00Z"
    }
  ]
}
```

#### `GET /tasks` — own task list

With a **user login**, returns the tasks of the **logged-in** user (assignee = current
user), paginated — this is the main task screen of an app. With an **API key** (no user),
returns **all** tasks of the tenant.

**Query parameters (optional):**

| Parameter      | Type     | Description |
|----------------|----------|-------------|
| `status`       | string   | Filter (repeatable), default `open,in_progress` |
| `due_before`   | datetime | Only tasks due before this time (e.g. "due today") |
| `priority`     | string   | Filter by priority |
| `page`/`limit` | int      | Pagination |

Response: same task object shape as above, plus `pagination`. When a task is linked to a
contact, it additionally carries a compact `contact` block (id, name, city) for list display.

#### `PATCH /tasks/{id}` — update a task

Lets a user complete or reschedule a task. **User login only.** With an API key this returns
`403 forbidden` ("write access requires a user token") — writes are always user-bound. A user
may only edit their **own** tasks (editing another user's task returns `403`); a task of
another tenant returns `404`.

**Request body (only the fields to change):**

```json
{
  "status": "done",
  "snoozed_until": "2026-06-16T08:00:00Z"
}
```

Editable fields in V1: `status`, `priority`, `due_at`, `snoozed_until`, `description`.
Setting `status: "done"` makes the API set `completed_at` automatically.

---

### 4.3 Documents

Documents attached to a contact. Each document carries a **visibility**: `internal` (staff
only) or `customer_ready` (shared with the customer). Files are stored privately; the API
returns a temporary signed `download_url`.

**Fixed value list — `visibility`:** `internal`, `customer_ready`.

#### `GET /contacts/{id}/documents` — documents of a contact (staff scope)

Returns all documents of the contact, both visibilities. Fields (curated): `id`, `title`,
`file_name`, `category`, `visibility`, `created_at`, `download_url`.

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Angebot Kundenhaus Adamello",
      "file_name": "angebot-adamello.pdf",
      "category": "Angebot",
      "visibility": "customer_ready",
      "created_at": "2026-06-14T08:00:00Z",
      "download_url": "https://...signed..."
    }
  ]
}
```

#### `GET /documents` — my shared documents (end-customer self scope)

Returns the documents shared with the calling customer — i.e. `visibility = customer_ready`
of the caller's own contact — each with a temporary signed `download_url`. Internal documents
are never returned, and `visibility` is omitted (customer-ready by definition). Response shape
as above, without the `visibility` field.

#### `PATCH /contacts/{id}/documents/{docId}/visibility` — set document visibility (staff, Admin/Manager)

Staff write, restricted to **Admin/Manager**. Switches a document between `internal` and
`customer_ready`; setting `customer_ready` shares it with the customer via `GET /documents`.

**Request body:**

```json
{ "visibility": "customer_ready" }
```

`visibility` must be `internal` or `customer_ready`. Returns the updated
`{ "id": "uuid", "visibility": "customer_ready" }`. A document of another tenant or contact
returns `404`.

> Uploading documents is staff-only and out of scope for the customer view; the customer read
> (`GET /documents`) is read-only.

---

### 4.4 Referrals

A referral links a **referrer** (an existing customer) to a person they referred. The
contract exposes **status, not identity** — the referred person's personal data is never
returned.

**Fixed value lists** (verified against the database):
- referral `status`: `registered`, `journey_started`, `contracted`, `paid`
- `payout_status`: `offen`, `faellig`, `ausgezahlt`, `storniert`

#### `GET /me/referral-link` — the caller's personal referral link (self scope)

```json
{ "slug": "doro-k", "url": "https://buedenbender-hausbau.de/tools/r/doro-k" }
```

#### `GET /referrals` — my referrals (end-customer self scope)

Returns the caller's own referrals (as referrer), **status only**. A data-protection threshold
applies to the level of detail: with exactly one active referral, only the coarse
`payout_status` is returned; the finer `status` is included only when several referrals are
active **or** the referred person has explicitly consented. Names or contact data of the
referred person are never returned.

```json
{
  "data": [
    {
      "id": "uuid",
      "payout_status": "offen",
      "status": "registered",
      "payout_amount_eur": 1000
    }
  ]
}
```

#### `POST /referrals` — submit a referral code (end-customer self scope)

The referral intake **is a live self-scope endpoint**. The caller is the **referred person**
(Geworbener), submitting the **referrer's** (Werber) code.

**Request body:**

```json
{ "referral_code": "doro-k", "detail_consent": false }
```

- `referral_code` (string, **required**) — the referrer's code.
- `detail_consent` (bool, optional) — the referred person's consent to expose the finer
  `status` to the referrer.

**Behaviour.** The referrer (Werber) is resolved via `user_profiles.referral_code`. The
referred contact is created/linked and a `contact_relationships` row is written with
`relationship_type = "empfehlung"` (from = Werber, to = Geworbener), idempotently via
`ON CONFLICT ON CONSTRAINT unique_relationship DO NOTHING`; a `referral_details` row is then
written via `relationship_id`. `public.referrals` is deprecated and left untouched.

**Response** — `201` when the referral was newly created, `200` when it already existed:

```json
{
  "id": "uuid",
  "validity": "gueltig",
  "payout_status": "offen",
  "created": true
}
```

`validity` is one of `gueltig`, `bestehend`, `ungeklaert`; `payout_status` as in the fixed
list above; `created` is `true` on a fresh `201`, `false` on an existing `200`.

---

### 4.5 Open-house requests

A customer can offer to open their home for a viewing. This is a **non-binding expression of
interest** that creates a review task for the responsible staff member; it is not
self-service, and the customer never enters visitor data.

#### `POST /open-house-requests` — express interest (self scope)

**Request body (all optional):**

```json
{ "timeframe": "fruehling", "message": "Wir wuerden gern unser Haus oeffnen." }
```

Creates a review task assigned to the responsible person and returns `202` with the request in
status `pending`. `timeframe` is a free grouping (e.g. `fruehling`, `sommer`, `herbst`,
`flexibel`).

#### `GET /open-house-requests` — my requests (self scope)

Returns the caller's own open-house requests and their status (`pending`, `confirmed`,
`declined`, `done`).

---

### 4.6 Chat

A single conversation per contact between the customer and the **team** — the assigned
Hausberater plus the supporting **Online-Team**. One thread per contact: the customer has
exactly one thread (their own); staff see all threads of the tenant.

**Message sender classes** (`sender_kind`): `customer`, `advisor` (the assigned Hausberater),
`online_team` (a supporting team member), `system` (automated note). The client renders the
Online-Team suffix ("Name (Online-Team)") from `sender_kind`; the assigned advisor shows
without a suffix.

**Message object:**

```json
{
  "id": "uuid",
  "thread_id": "uuid",
  "created_at": "2026-07-14T14:09:00Z",
  "body": "Guten Tag Herr Schilling, ich habe Ihnen die Baubeschreibung bereitgestellt.",
  "sender_kind": "online_team",
  "sender": { "id": "uuid", "name": "Lena Vogt", "avatar_url": "https://..." }
}
```

`sender` is `null` for `system` messages.

#### Self scope (customer)

- `GET /chat` — my thread meta: the participants (advisor + Online-Team members, each with
  name/avatar) and the unread count.
- `GET /chat/messages` — my messages, paginated (`page`/`limit`).
- `POST /chat/messages` — send a message. Body `{ "body": "..." }`; `sender_kind` is `customer`.
- `POST /chat/read` — mark my thread read up to now.

#### Staff scope

- `GET /chat/threads` — the inbox: threads across contacts, each with a contact summary, last
  message, unread count and assignment. Query: `status`, `unread` (bool), `page`/`limit`.
- `GET /chat/threads/{id}/messages` — a thread's messages, paginated.
- `POST /chat/threads/{id}/messages` — reply. Body `{ "body": "..." }`. `sender_kind` is
  `advisor` when the caller is the contact's assigned Hausberater, otherwise `online_team`.
- `PATCH /chat/threads/{id}/read` — mark the thread read.
- `GET /chat/threads/{id}/canned-replies` — contextual suggested replies for this thread (based
  on the last customer message / contact status): `{ "data": [ { "id": "uuid", "text": "...",
  "category": "..." } ] }`. A chosen reply is placed into the input as editable text — it is not
  sent automatically.

> **Realtime.** History and sending go through these REST endpoints; **live delivery** is via
> Supabase Realtime — each client subscribes to new messages of its own thread (RLS-scoped), so
> messages appear instantly without polling. When the app is in the background, delivery is
> covered by push (§5). Typing/presence indicators, if wanted, use Realtime presence channels.

> **Backend note.** Built on the existing chat structure (`card_chat_threads` /
> `card_chat_messages`, generalized) plus additions: a message **sender** (`sender_user_id` +
> `sender_kind`), thread **read state**, the **Online-Team** (a functional `teams` /
> `team_members` group), and a small per-tenant **canned-replies** store. **This backend is now
> in place — all nine chat endpoints in this section are deployed and live.**

---

## 5. PHASE 2 — Notifications & push

Two parts, both scoped to the calling user: **in-app notifications** (already stored) and
**device push** (delivery to the phone).

### 5.1 In-app notifications

#### `GET /notifications` — my notifications

User login. Returns the caller's notifications, newest first, paginated. Fields (curated):
`id`, `type`, `title`, `body`, `link`, `is_read`, `created_at`.

**Query parameters (optional):** `unread` (bool — only unread), `page`/`limit`.

```json
{
  "data": [
    {
      "id": "uuid",
      "type": "task_assigned",
      "title": "Neue Aufgabe",
      "body": "Rueckruf fuer Familie Mustermann",
      "link": "/tasks/uuid",
      "is_read": false,
      "created_at": "2026-06-14T08:00:00Z"
    }
  ],
  "pagination": { "page": 1, "limit": 25, "total": 3 }
}
```

#### `PATCH /notifications/{id}` — mark as read

User login. Body `{ "is_read": true }`; sets `read_at`. A notification of another user returns
`404`.

### 5.2 Device push

To deliver push to a phone, the app registers its device after login.

#### `POST /devices` — register a device

User login. Body `{ "push_token": "...", "platform": "ios" }` (`platform`: `ios` | `android`).
Idempotent per token; stored per user.

#### `DELETE /devices/{token}` — unregister a device

User login. On logout, or when a token becomes invalid.

> **Backend note.** The per-user device-token store is the piece to build for Phase 2 (today a
> single `push_token` exists per user). The **push provider** is an open decision — Firebase
> Cloud Messaging (FCM) for both platforms, or APNs (iOS) + FCM (Android) — and determines the
> token format the app sends. To decide before build.

### 5.3 Channel preferences

#### `GET /notification-preferences` · `PUT /notification-preferences`

User login. The caller's channel preferences. Fields: `in_app` (bool), `email` (bool),
`email_digest` (`instant` | `hourly` | `daily` | `off`), `muted_types` (array of notification `type` values to
suppress). `PUT` replaces the caller's preferences.

---

## 6. Roadmap beyond Phase 2 (concept — plan the architecture, do not build yet)

These are documented so the app architecture can anticipate them. None of them is final;
each is delivered in a scoped later version.

### 6.1 Card endpoints (the third core area)

The integrated app has three core areas: Contacts, Todos and **Card**. The Card is a
field-facing contact hub per sales rep (a branded mini-page that bundles communication entry
points and feeds the CRM timeline). The backend for the Card (its own tables, event logging,
web-chat) is being built now (separate strand). The **API endpoints for the Card** — for a
client to read/manage a rep's card config and see card-driven contacts/activities — will be
specified here once the Card backend is in place. Plan the app for three areas, not two; the
Card screens consume Card endpoints under the same `/api/v1/` contract.

### 6.2 End-customer (self) scope — first endpoints introduced

The end-customer (self) scope is now part of the contract (see §2 "Authorization scopes"): a
person with an app login, constrained to their own records. The first self-scope endpoints are
specified above — the `advisor` block on `/me` (§4.0), Documents (§4.3), Referrals (§4.4) and
Open-house requests (§4.5) and Chat (§4.6). Further end-customer resources (consents /
notification preferences, favorites, journey / milestones) follow as additional resource
sections in scoped later versions, on the same host, error shape and conventions.

### 6.3 Other planned additions

Activities/history per contact, creating a contact, additional filters, granular API-key
scopes (V1 keys carry full read access; finer scopes are a later extension), and
tenant-specific field/module variations — all added in clearly scoped later versions.

---

## 7. What we need from you / next steps

1. **Feedback on the Phase 1 spec:** do the fields match what the app screens need? Anything
   missing for the contact or task views?
2. **App technology:** which platform / framework is planned (native iOS/Android, React
   Native, Flutter)? Mainly relevant for the push path in Phase 2 and the Card screens.
3. **Base URL:** all work targets `https://api.tybird.com/api/v1/` (note the `/api/v1/`
   path). The `meinbbh...` host is **not** the API base — do not use it.
4. **Credentials:** provided per the chosen auth model — a user account (Model 1) or a
   tenant API key (Model 2). For a user login we also provide the Supabase URL + anon key for
   the login flow; login runs at `https://api.tybird.com/auth/v1/token`.

> This specification is deliberately the smallest sensible starting point. Everything else
> (Card endpoints, activities/history, end-customer access, creating a contact, additional
> filters, granular key scopes) is added in clearly scoped later versions.

---

## 8. Changelog

What changed between revisions of this spec. Newest first.

- **2026-07-24** — Vertrag an den Ist-Stand angeglichen: POST /referrals (relationship_type empfehlung) dokumentiert; email_digest um hourly/off ergaenzt; documents visibility-PATCH aufgenommen; Chat und Self-Scope als live markiert. Offen: devices/Push (Phase 2).
- **2026-07-20** — Added **Chat** (§4.6): a per-contact conversation for the self and staff
  scopes — sender classes (`customer`/`advisor`/`online_team`/`system`), a staff inbox,
  contextual canned replies, and a Realtime note. The backend (chat tables + message sender +
  read state, the Online-Team group, the canned-replies store) is still to build.
- **2026-07-20** — Introduced the **end-customer (self) scope** (§2 "Authorization scopes"):
  a caller constrained to their own records, alongside the existing staff scope. Added the
  first self-scope resources — the `advisor` block on `/me` (§4.0), **Documents** (§4.3, incl.
  the `internal`/`customer_ready` visibility and the customer's read of shared documents),
  **Referrals** (§4.4, status-only + personal link), and **Open-house requests** (§4.5). Also
  specified **Phase 2 — Notifications & push** (§5): `GET`/`PATCH /notifications`,
  `POST`/`DELETE /devices`, and `GET`/`PUT /notification-preferences` (the device-token store
  and push provider are still to build). Additive; the staff Contacts/Tasks endpoints and JSON
  shapes are unchanged.
- **2026-06-26** — Added `GET /me` (§4.0): returns the active tenant (id, slug, name, `brand`
  limited to `accent`/`font`/`logo_url`, `enabled_modules`) for app branding, plus the
  caller's own identity on a user login (`user` is `null` for an API key). Read-only;
  existing endpoints and JSON shapes unchanged.
- **2026-06-23** — Host consolidation. The single base URL is now
  `https://api.tybird.com/api/v1/` (corrected from `/v1/`). The `meinbbh...` host was
  removed as an API base. Added the Quick test (smoke test) in §2 with a Windows PowerShell
  note. Documented `404` as a wrong-path indicator. Authentication, endpoints and JSON shapes
  unchanged.
- **2026-06-21** — Tenant scoping enforced on every read and write (previously marked "in
  progress"). Two authentication models introduced: user login (JWT, Model 1) and
  tenant-scoped API key (Model 2). Phase-1 endpoints and JSON shapes unchanged.
- **Version 1 (initial)** — First Phase-1 specification: Contacts and Tasks endpoints;
  Phase-2 push notifications as a concept; Card endpoints and the end-customer access path on
  the roadmap.
