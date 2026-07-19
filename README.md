# tybird API

The official, versioned **API contract** of the tybird platform. Apps - the "Mein Buedenbender"
end-customer app, the internal CRM, and future partner apps - build exclusively against this
contract, not against the internal database schema. This is the HubSpot model: one platform with
one official API, and any number of apps on top.

> Status: **v0.1 (draft)** - the first curated surface, covering the end-customer app.
> Spec-first: the contract is deliberately defined; the runtime behind it (Supabase - curated
> RPCs plus the `api` edge function, with PostgREST/RLS underneath) implements exactly these
> signatures.

## Contents

```
tybird-api/
├── openapi/
│   └── tybird.v0.1.yaml        # the contract (OpenAPI 3.1, versioned)
├── .github/workflows/
│   └── publish.yml             # lint + Redoc build -> GitHub Pages (on merge to main)
├── redocly.yaml                # lint/build configuration
├── index.html                  # local preview (Redoc CDN, no build)
├── CHANGELOG.md
└── README.md
```

## Principles

- **One backend, no second one.** The contract is the public surface of one platform (Supabase,
  BBH as a tenant). No app gets its own backend/DB.
- **Curated.** Only the public contract surface goes into the published spec - no internal
  schema, no secrets, no admin RPCs. GitHub Pages is public.
- **Versioned (SemVer).** Breaking changes = new major version + new spec file
  (`tybird.vX.Y.yaml`). Internal schema changes only break apps when the CONTRACT changes - a
  deliberate, versioned decision.
- **Tenant + row view server-side.** The tenant comes from the JWT, never as a parameter. An end
  customer only ever sees their own data (RLS). End customers receive their tenant via an
  `endkunde` tenant membership.
- **Language:** the contract and its docs are English. User-facing copy inside the apps is
  German ("Sie") - that is a frontend concern, not part of this contract.

## Publishing (GitHub Pages)

On every merge to `main` that touches `openapi/**`, `publish.yml` runs:
1. **Lint** the spec (`@redocly/cli lint`) - an invalid contract fails the build.
2. **Build** the static docs (`@redocly/cli build-docs`, Redoc).
3. **Deploy** to GitHub Pages.

One-time setup: create the `tybird-api` repo, then Settings -> Pages -> Source = "GitHub Actions".
After that, every merge publishes automatically.

Local preview without a build: open `index.html` in a browser.

## Extending the contract

1. Add the endpoint/schema in `openapi/tybird.v0.1.yaml` (spec-first).
2. Lint locally: `npx @redocly/cli lint openapi/tybird.v0.1.yaml`.
3. Open a PR. Merge -> Pages updates automatically.
4. Breaking change? Create a new version and record it in `CHANGELOG.md`.

## Roadmap

- **CRM surface:** v0.1 covers the end-customer surface; the broader CRM/admin surface
  (including the territory-management RPCs) grows as additional tags in the same contract.
- **Auth (P0, verified):** end customers get the tenant via an `endkunde` membership; RLS
  enforces the self-view. Two RLS gaps to close before go-live (favorites insert/delete,
  appointments consumer read).
- **Runtime binding:** the `api` edge function routes contract paths to the curated RPCs/tables.
