# tybird API

**Single source of truth** for the tybird platform API. This repository *is* the API contract - there is no second place it lives.

- **The spec:** [`API.md`](API.md) - the canonical, human-readable specification (Contacts, Tasks, `/me`, and the roadmap).
- **Published (rendered):** https://billiton-gmbh.github.io/tybird-api/
- **One platform, one API:** the internal tybird CRM (staff app, built by Tam) and the end-customer app "Mein Buedenbender" build against this same contract.

## How this repo works

- `API.md` is the truth. Edit it here; nothing is maintained in two places.
- Every merge to `main` that touches `API.md` runs GitHub Actions, which renders it and publishes to GitHub Pages.
- The changelog lives inside `API.md` (its "Changelog" section).

## Scope (tracked inside API.md)

- **Phase 1 (live):** Contacts, Tasks, `/me`.
- **Phase 2:** push notifications (device registration).
- **Later:** Card endpoints; the end-customer access path for "Mein Buedenbender" - a scoped, end-customer role against this same contract.
