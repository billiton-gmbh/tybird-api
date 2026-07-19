# Changelog - tybird API

Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning per SemVer.

## [0.1.0] - 2026-07-19
### Added
- First curated contract (OpenAPI 3.1, English) covering the "Mein Buedenbender" end-customer app.
- Areas: Identity & Phase (`/me`, `/me/phase`), Referrals incl. the anonymity threshold
  (`/referrals`, `/referrals/link`, `/referrals/{id}/detail-consent`), Appointments
  (`/appointments`), Notifications & consents (`/notification-preferences`, `/consents`),
  Journey & Open House (`/journey`, `/open-house/interest`), Discover (`/houses`, `/favorites`),
  Documents (`/documents`).
- Auto-publish to GitHub Pages (Redoc) via GitHub Actions.

### Notes
- Auth model verified (P0): end customers get the tenant via an `endkunde` membership; RLS
  enforces the self-view. Two RLS gaps tracked before go-live.
- English is the contract default; no German version.
