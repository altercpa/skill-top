# User API

The AlterCPA TOP public API lets you manage bots and campaigns, pull statistics,
export leads, and handle billing — programmatically. It exposes the **same surface
the dashboard uses**, so everything you can do in the UI you can script.

**Base URL:** `https://my.altercpa.top/api`

## Quick start

```bash
curl "https://my.altercpa.top/api/profile/info.json?id=42-d3adb33f…"
```

```json
{ "status": "ok", "data": { "id": 42, "name": "John Doe", … } }
```

Replace `42-d3adb33f…` with your `{user_id}-{user_api}` key from
[Profile → API key](https://docs.altercpa.top/en/guide/profile/). See
[Authentication & conventions](authentication.md) for all auth options and
the full response envelope.

## Sections

| Section | What you can do |
|---|---|
| [Authentication & conventions](authentication.md) | Auth methods, JSON envelope, common errors |
| [Profile & tracker](profile.md) | Read account info, rotate API key, set global postbacks, connect a tracker |
| [Bots](bots.md) | List, connect, fetch admin chats, disconnect |
| [Campaigns, screens & media](campaigns.md) | Full CRUD, fetch deep link, pause/resume, welcome screens and media, CAS guard and group cleanup |
| [Broadcasts](broadcasts.md) | Configure triggered follow-ups, create and run manual broadcasts |
| [CRM: teams & operators](crm.md) | Teams, operators, templates, custom events and CRM stats |
| [Leads, journal & stats](leads-stats.md) | Per-status summary, export conversions to CSV, browse chat activity, campaign leaderboard |
| [Transfer](transfer.md) | Hand bots or campaigns to another account |
| [Tariffs & billing](billing.md) | List plans, start a payment, resume and cancel payments, view payment history |
