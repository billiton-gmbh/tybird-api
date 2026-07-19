# tybird CRM App — API Specification V1

**Owner:** billiton internet services GmbH
**Repository path:** `billiton-gmbh/tybird-api/API.md` (this file — the single source of truth)
**Last updated:** 2026-06-26
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
| Authorization      | Internal staff scope only in V1. No end-customer access. |
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

## 5. PHASE 2 — Push notifications (concept, not final)

Goal: when a task is assigned to a user or becomes due, the user receives a push
notification on the device.

**Current state of the backend:** an internal `notifications` structure already exists (for
in-app notifications: title, body, read status, link). What is **missing** and must be built
for real device push is **device registration** (device token management). We build this once
Phase 1 is in place.

**Likely endpoints (for early planning in the app architecture):**

- `POST /devices` — register a device: after login the app sends the push token (FCM/APNs)
  and the platform (`ios`/`android`). We store it per user.
- `DELETE /devices/{token}` — unregister a device (logout / token invalid).
- `GET /notifications` — in-app notification list (read/unread).
- `PATCH /notifications/{id}` — mark as read.

**Open decision for Phase 2 (to decide together):** push provider — Firebase Cloud Messaging
(FCM) for both platforms, or APNs (iOS) plus FCM (Android) separately. This affects which
token format the app delivers.

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

### 6.2 End-customer access path (future, not in V1)

Today this API is **staff-only** (internal staff accounts; end-customer accounts are
rejected). A separate end-customer-facing app ("Mein Büdenbender") is planned to build
against this same API in the future. That requires a distinct, end-customer-scoped access
path (a different role/scope, narrower data) — it is **not** part of V1 and is called out
here only so the contract's staff-only boundary is explicit. The end-customer scope will be
specified separately before it is built.

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
