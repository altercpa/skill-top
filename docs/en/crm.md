# CRM: teams & operators

Advertiser-side CRM management: per-bot CRM activation, operator teams with their
permissions, the operators themselves, canned-reply templates and sets, custom events
(operator-fired postbacks), statistics and dialog oversight. This is the account
owner's API (`/api/crm/*`). The operator workplace lives separately on
crm.altercpa.top — its API (`/op/*`) is **not** covered by this reference.

All responses are HTTP 200 with `application/json`; success or failure is indicated in
the body, not in the HTTP status. Time is returned in **unix seconds** (`0` = unset).

<a id="gating"></a>

## Authorization and tariff gates

All 23 endpoints use API-key or dashboard-session authentication
(`?id={uid}-{key}`) and do **not** require an active paid subscription — so an owner
with a lapsed subscription can still manage CRM and **turn it off** per bot in order to
wind down cleanly.

The CRM feature on your tariff is checked on the server on every mutating action, as
well as on the "heavy" GETs — statistics and the dialog list. The lightweight lists
(`bot/list`, `team/list`, `operator/list`, `template/list`, `event/list`) do **not**
require the feature — only ownership of the resource. Turning CRM on per bot
(`bot.json` with `crm=1`) requires the feature; **turning it off is always allowed**.

Your operator-seat limit is checked when creating an operator and when enabling one
(`status=0`); the seat limit is enforced atomically, and when there is no room you get
`error: limit`.

Parameters are accepted as a query string, a urlencoded body, or a JSON body (JSON
takes priority); lists (`bots`, `sets`) accept a JSON array or a comma-separated string.

Cross-cutting error codes: `key` (missing/invalid token — the client logs out),
`ban` (user blocked), `access` (resource not yours / not found),
`func` (bad arguments), `db` (database error), `limit` (seat cap reached).
Codes specific to this page: `crm` (the CRM feature is not on the tariff — the front
upsells), `inuse` (the team still has operators), `media` (bad media path/type). CRM
tariffs are described in the [tariffs & billing](billing.md) section.

---

<a id="crm-bot-list"></a>

## Bots with CRM

`GET /api/crm/bot/list.json`

A list of your non-deleted bots with their CRM state (per-bot CRM activation). Ordered
by bot id descending. Deleted bots are excluded. Does not require the CRM feature.

| Parameter | Description |
|---|---|

```json
{
  "status": "ok",
  "data": [
    { "id": 42, "username": "myshopbot", "name": "My Shop", "crm": true },
    { "id": 37, "username": "leadsbot", "name": "Leads", "crm": false }
  ]
}
```

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="crm-bot"></a>

## Enable/disable CRM per bot

`POST /api/crm/bot.json`

Toggles per-bot CRM activation for one of your bots. **Enabling** (`crm=1`) requires
the CRM feature; **disabling** (`crm=0`) is always allowed — it is the "emergency exit"
for an owner with a lapsed subscription. The change immediately refreshes the bot's
settings on our backend.

| Parameter | Required | Description |
|---|---|---|
| `bot` | ✓ | Bot id; must be yours and not deleted. `0`/absent → `func` |
| `crm` | ✓ | Desired state; must be present. Truthy: `1`/`true`/`yes`/`on` |

```json
{ "status": "ok", "bot": 42, "crm": true }
```

| Error | Meaning |
|---|---|
| `func` | `bot` not given or `crm` absent |
| `access` | Bot not yours or deleted |
| `crm` | Enabling without the CRM feature on the tariff |
| `db` | Database error |

---

<a id="crm-team-list"></a>

## List teams

`GET /api/crm/team/list.json`

Your teams with their six permission flags and the list of allowed bots (allowlist).
Ordered by team id. Does not require the CRM feature.

Permission flags: `leads_view` (see the lead card), `leads_edit` (edit the lead),
`confirm` (approve a join request, `approveChatJoinRequest`),
`events` (fire custom events), `reply` (send messages),
`sticky` (pin a dialog to an operator).

| Parameter | Description |
|---|---|

```json
{
  "status": "ok",
  "data": [
    {
      "id": 3,
      "name": "Поддержка",
      "leads_view": true,
      "leads_edit": true,
      "confirm": true,
      "events": false,
      "reply": true,
      "sticky": false,
      "bots": [42, 37],
      "created": 1718900000
    }
  ]
}
```

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="crm-team"></a>

## Create or edit a team

`POST /api/crm/team.json`

Creates a team (when `id=0`) or changes its name and six permission flags. On edit, a
permission downgrade applies to the team's operators immediately.

| Parameter | Required | Description |
|---|---|---|
| `id` | | `0`/absent — create; otherwise — edit (the team must be yours) |
| `name` | | Team name |
| `leads_view` | | Permission: see the lead |
| `leads_edit` | | Permission: edit the lead |
| `confirm` | | Permission: approve a join request |
| `events` | | Permission: fire custom events |
| `reply` | | Permission: send messages |
| `sticky` | | Permission: dialog pinning |

```json
{ "status": "ok", "id": 3 }
```

On create `id` is the new identifier; on edit it is the `id` you passed in.

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Editing a team that is not yours |
| `db` | Database error |

---

<a id="crm-team-del"></a>

## Delete a team

`POST /api/crm/team/del.json`

Deletes the team together with its bot allowlist and template-set shares. **Refuses**
if the team still has operators — reassign or delete them first.

| Parameter | Required | Description |
|---|---|---|
| `team` | ✓ | Team id; must be yours |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Team not yours |
| `inuse` | The team still has operators — reassign/delete them |
| `db` | Database error |

---

<a id="crm-team-bots"></a>

## Replace a team's bots

`POST /api/crm/team/bots.json`

Fully replaces the team's bot allowlist with the supplied set. Foreign and deleted
bots are silently dropped; the response contains the validated, persisted subset (order
is preserved). An empty/absent list clears the allowlist.

| Parameter | Required | Description |
|---|---|---|
| `team` | ✓ | Team id; must be yours |
| `bots` | | JSON array or comma-separated string of bot ids |

```json
{ "status": "ok", "bots": [42, 37] }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Team not yours |
| `db` | Database error |

---

<a id="crm-team-sets"></a>

## Replace a team's sets

`POST /api/crm/team/sets.json`

Fully replaces the template sets shared to the team. Foreign sets are silently
dropped; the response contains the validated, persisted subset. An empty/absent list
removes all shares.

| Parameter | Required | Description |
|---|---|---|
| `team` | ✓ | Team id; must be yours |
| `sets` | | JSON array or comma-separated string of set ids |

```json
{ "status": "ok", "sets": [5, 8] }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Team not yours |
| `db` | Database error |

---

<a id="crm-operator-list"></a>

## List operators

`GET /api/crm/operator/list.json`

The account's full operator roster plus seat usage. The secret is **never** returned —
the plaintext token is shown only once, on create or reset. `used` counts active
operators (`status=0`), `cap` is your operator-seat limit. Does not require the CRM
feature.

`status`: `0` active, `1` disabled. `lead` — permission to work with leads,
`manage` — management permission (implies `lead`).

| Parameter | Description |
|---|---|

```json
{
  "status": "ok",
  "operators": [
    {
      "id": 11,
      "name": "Анна",
      "login": "anna",
      "team": 3,
      "status": 0,
      "lead": true,
      "manage": false,
      "seen": 1718950000,
      "created": 1718900000
    }
  ],
  "seats": { "used": 1, "cap": 15 }
}
```

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="crm-operator"></a>

## Create or edit an operator

`POST /api/crm/operator.json`

Creates an operator (when `id=0`) or changes its name, login, team and role flags.
**On create** a one-time plaintext `token` (64 characters) is returned — it is shown
once and stored only hashed; the seat limit is enforced atomically. On
edit the token is not returned. `manage` implies `lead`.

| Parameter | Required | Description |
|---|---|---|
| `id` | | `0`/absent — create; otherwise — edit (the operator must be yours) |
| `name` | | Display name |
| `login` | | Login label |
| `team` | | Team id; if not `0`, must be yours. `0` — no team |
| `manage` | | Role: management (implies `lead`) |
| `lead` | | Role: working with leads; forced `true` when `manage` is set |

Create:

```json
{ "status": "ok", "id": 11, "token": "9f3c…64-character-secret…b2a1" }
```

Edit:

```json
{ "status": "ok", "id": 11 }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Team not yours, or editing an operator that is not yours |
| `limit` | Seat cap reached (on create) |
| `db` | Database error |

---

<a id="crm-operator-status"></a>

## Enable/disable an operator

`POST /api/crm/operator/status.json`

Enables (`status=0`) or disables (`status=1`) an operator. **Disabling** returns their
dialogs to the shared pool and removes their sessions. **Enabling** re-claims a seat
(the seat limit is enforced atomically).

| Parameter | Required | Description |
|---|---|---|
| `operator` | ✓ | Operator id; must be yours |
| `status` | ✓ | `1` — disable (return dialogs to the pool + remove sessions); otherwise (`0`) — enable |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Operator not yours |
| `limit` | No free seat (on enable) |
| `db` | Database error |

---

<a id="crm-operator-reset"></a>

## Reset operator access

`POST /api/crm/operator/reset.json`

Reissues the operator's secret: returns a new one-time plaintext `token`
(64 characters, shown once), the old access link is invalidated, and all of the
operator's sessions are removed.

| Parameter | Required | Description |
|---|---|---|
| `operator` | ✓ | Operator id; must be yours |

```json
{ "status": "ok", "token": "1a7e…new-64-character-secret…c4d9" }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Operator not yours |
| `db` | Database error |

---

<a id="crm-operator-del"></a>

## Delete an operator

`POST /api/crm/operator/del.json`

Deletes the operator, returning their dialogs to the shared pool and removing their
sessions.

| Parameter | Required | Description |
|---|---|---|
| `operator` | ✓ | Operator id; must be yours |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Operator not yours |
| `db` | Database error |

---

<a id="crm-template-list"></a>

## List templates and sets

`GET /api/crm/template/list.json`

Returns both the sets (`sets`, including the teams each is shared to) and a flat array
of all templates (`templates`, each carrying its own `set`). Sets are ordered by
set id; templates by set, sort order, and id. Does not require the CRM feature.

`media_type`: `0` none, `1` photo, `2` video, `3` document.

| Parameter | Description |
|---|---|

```json
{
  "status": "ok",
  "sets": [
    { "id": 5, "name": "Приветствия", "teams": [3], "created": 1718900000 }
  ],
  "templates": [
    {
      "id": 21,
      "set": 5,
      "name": "Привет",
      "text": "Здравствуйте! Чем помочь?",
      "media": "",
      "media_type": 0,
      "sort": 0
    }
  ]
}
```

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="crm-template"></a>

## Create or edit a template

`POST /api/crm/template.json`

Creates (when `id=0`) or edits a single canned-reply template: the text and optional
media, optionally assigned to a set. The media is validated as belonging to you;
an empty `media` resets `media_type` to `0`.

| Parameter | Required | Description |
|---|---|---|
| `id` | | `0`/absent — create; otherwise — edit (the template must be yours) |
| `set` | | Set id; if not `0`, must be yours. `0` — no set |
| `name` | | Template label |
| `text` | | Reply body |
| `media` | | Path to your media from [`media/upload`](campaigns.md#media-upload); `""` if none |
| `media_type` | | `1` photo, `2` video, `3` document (when `media` is set) |
| `sort` | | Order within the set |

```json
{ "status": "ok", "id": 21 }
```

On create `id` is the new identifier; on edit it is the `id` you passed in.

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Set not yours, or editing a template that is not yours |
| `media` | Bad media path or `media_type` |
| `db` | Database error |

---

<a id="crm-template-del"></a>

## Delete a template

`POST /api/crm/template/del.json`

Deletes a single template (deleted only if it belongs to you). There is no
separate ownership pre-check: a foreign or non-existent `id` is a silent no-op, the
response is still `ok`.

| Parameter | Required | Description |
|---|---|---|
| `template` | ✓ | Template id (deleted only if it belongs to you) |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `db` | Database error |

---

<a id="crm-set"></a>

## Create or rename a set

`POST /api/crm/set.json`

Creates a template set (when `id=0`) or renames it.

| Parameter | Required | Description |
|---|---|---|
| `id` | | `0`/absent — create; otherwise — rename (the set must be yours) |
| `name` | | Set name |

```json
{ "status": "ok", "id": 5 }
```

On create `id` is the new identifier; on rename it is the `id` you passed in.

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Renaming a set that is not yours |
| `db` | Database error |

---

<a id="crm-set-del"></a>

## Delete a set

`POST /api/crm/set/del.json`

Deletes the set and its team shares, **ungrouping** its templates: they are removed
from the set, the content itself is kept.

| Parameter | Required | Description |
|---|---|---|
| `set` | ✓ | Set id; must be yours |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Set not yours |
| `db` | Database error |

---

<a id="crm-event-list"></a>

## List custom events

`GET /api/crm/event/list.json`

The custom-event definitions of one bot (postbacks that operators fire), both active
and disabled. Ordered by sort order and event id. The gate is **bot ownership only**;
this list does not require the CRM feature.

`status`: `0` active, `1` disabled.

| Parameter | Required | Description |
|---|---|---|
| `bot` | ✓ | Bot id; must be yours. `0`/absent → `func` |

```json
{
  "status": "ok",
  "data": [
    {
      "id": 7,
      "bot": 42,
      "name": "Оплата получена",
      "desc": "Менеджер подтвердил оплату",
      "url": "https://track.example.com/postback?click={click}&goal=paid",
      "sort": 0,
      "status": 0,
      "created": 1718900000
    }
  ]
}
```

| Error | Meaning |
|---|---|
| `func` | `bot` not given (`0`) |
| `access` | Bot not yours |
| `db` | Database error |

---

<a id="crm-event"></a>

## Create or edit an event

`POST /api/crm/event.json`

Creates (when `id=0`) or edits a custom-event definition. A custom event is a
**postback**, so `url` is mandatory (otherwise `func`). On create, bot ownership is
checked; `bot` is **immutable** on edit. You can disable it via either `disabled` or
`status=1`.

| Parameter | Required | Description |
|---|---|---|
| `id` | | `0`/absent — create; otherwise — edit (the event must be yours) |
| `bot` | ✓ | Bot id (used/checked only on create; immutable on edit) |
| `name` | | Event label shown to operators |
| `desc` | | Description |
| `url` | ✓ | Postback URL template; empty → `func` |
| `sort` | | Order |
| `disabled` | | If truthy → `status=1` (disabled) |
| `status` | | Alternative: `status=1` also disables |

```json
{ "status": "ok", "id": 7 }
```

On create `id` is the new identifier; on edit it is the `id` you passed in.

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `func` | Empty `url` |
| `access` | Bot not yours (on create), or editing an event that is not yours |
| `db` | Database error |

---

<a id="crm-event-del"></a>

## Delete an event

`POST /api/crm/event/del.json`

Deletes the event definition (deleted only if it belongs to you). The history of
fired events is kept. There is no separate ownership pre-check: a
foreign `id` is a silent no-op.

| Parameter | Required | Description |
|---|---|---|
| `event` | ✓ | Event id (deleted only if it belongs to you) |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `db` | Database error |

---

<a id="crm-stats"></a>

## CRM statistics

`GET /api/crm/stats.json`

Account-wide CRM statistics for a date range: volume, queue health, operator presence,
and per-operator workload (across all the owner's bots, teams and operators). Defaults
to the last 7 days.

In `volume`, system notes are excluded (only regular messages are counted). In
`queue`: `open` — all open dialogs; `unclaimed` — in the pool, no operator; `pending` —
open dialogs awaiting an operator reply (claimed or not): this is the actionable backlog
the interface warns on (not `unclaimed`); `oldest_unanswered` — the time of the oldest
unanswered inbound message (by the same "awaiting reply" logic, not by unread).
`presence.online` — operators active within the last 5 minutes.

| Parameter | Required | Description |
|---|---|---|
| `from` | | Range start (unix); default `now − 7 days` |
| `to` | | Range end (unix); default now |

```json
{
  "status": "ok",
  "from": 1718300000,
  "to": 1718900000,
  "volume": { "dialogs": 120, "messages_in": 540, "messages_out": 610 },
  "queue": { "open": 8, "unclaimed": 3, "pending": 5, "oldest_unanswered": 1718890000 },
  "presence": { "online": 2 },
  "operators": [
    {
      "id": 11,
      "name": "Анна",
      "messages_sent": 210,
      "dialogs_handled": 64,
      "confirms": 18,
      "leads_edited": 12,
      "events_fired": 9
    }
  ]
}
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `db` | Database error |

---

<a id="crm-dialog-list"></a>

## Dialog oversight

`GET /api/crm/dialog/list.json`

Read-only: a list of the account's open dialogs, newest first, paged by the `before`
cursor (the dialog cursor). Message bodies are not returned (oversight, not chat).
Page size is 50; `more=true` when the page is full.

`operator`: `0` — the dialog is in the pool (in which case `operator_name` is empty).
`pending`: `1` — the dialog is awaiting an operator reply. `last_in`/`last_out` — the
time of the last inbound (subscriber) and last operator reply (`0` — never answered);
`created` — the dialog start time.

| Parameter | Required | Description |
|---|---|---|
| `before` | | The dialog cursor: returns dialogs with `last_id` less than the given value. `0`/absent — first page |
| `filter` | | Status filter (applied server-side across the whole account): `pending` — awaiting reply; `unclaimed` — in the pool, no operator; `assigned` — claimed by an operator |
| `bot` | | Restrict to a single bot (its ID) |

```json
{
  "status": "ok",
  "dialogs": [
    {
      "id": 301,
      "bot": 42,
      "subscriber": 123456789,
      "lead": 5001,
      "name": "Иван",
      "username": "ivan",
      "operator": 11,
      "operator_name": "Анна",
      "unread": 2,
      "pending": 1,
      "last_id": 9870,
      "last_in": 1718895000,
      "last_out": 1718896000,
      "created": 1718800000
    }
  ],
  "more": false
}
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `db` | Database error |

---

<a id="crm-dialog-assign"></a>

## Reassign a dialog

`POST /api/crm/dialog/assign.json`

Reassigns a dialog to one of your operators, or force-returns it to the shared pool
(`operator=0`). Writes a system note, and operator inboxes refresh immediately.

| Parameter | Required | Description |
|---|---|---|
| `dialog` | ✓ | Dialog id; must belong to you |
| `operator` | | Target operator (must be yours); `0` or absent — return to the pool |

```json
{ "status": "ok", "operator": 11 }
```

| Error | Meaning |
|---|---|
| `crm` | The CRM feature is not on the tariff |
| `access` | Dialog not yours / not found, or the target operator is not yours |
| `db` | Database error |
