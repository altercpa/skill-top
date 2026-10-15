# Broadcasts

Configure **triggered** per-status nudges on a campaign, and create and run **manual**
bulk broadcasts. For the concepts see the [broadcasts guide](https://docs.altercpa.top/en/guide/broadcasts/);
this page is the API reference.

All broadcast endpoints require an active **paid** session (otherwise `error: unpaid`).
The **broadcasts** tariff feature is checked only by the actions that create sends:
`add`, `edit`, `start` of manual broadcasts and **enabling** a triggered block (`on=1`) —
without it they return `error: broadcasts`. Viewing and winding down already existing
broadcasts (`list`, `quota`, `get`, `pause`, `resume`, `reset`, `cancel`, `del`, `test`)
require no feature, so after a downgrade you can still oversee and gracefully finish
existing broadcasts.

Every response is HTTP 200 with `application/json`; success or error is reported in the
body. Times are unix seconds (`0` = unset).

---

<a id="camp-broadcast"></a>

## Get campaign blocks

`GET /api/camp/broadcast.json`

Message blocks by campaign and status. Five statuses, one block each:
`0` wait · `1` hold · `2` approve · `3` cancel · `4` trash. Returns all five
blocks; statuses without a block come back as a zero block (`on: 0`).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{
  "status": "ok",
  "data": [
    { "status": 0, "on": 1, "delay": 1800,
      "text": "Welcome! Tap the button below 👇",
      "media": "", "media_type": 0,
      "button": "Open", "link": "https://example.com/l?click={click}" },
    { "status": 1, "on": 0, "delay": 0, "text": "", "media": "", "media_type": 0, "button": "", "link": "" },
    { "status": 2, "on": 0, "delay": 0, "text": "", "media": "", "media_type": 0, "button": "", "link": "" },
    { "status": 3, "on": 0, "delay": 0, "text": "", "media": "", "media_type": 0, "button": "", "link": "" },
    { "status": 4, "on": 0, "delay": 0, "text": "", "media": "", "media_type": 0, "button": "", "link": "" }
  ]
}
```

`delay` is in **seconds** after the lead enters the status. `media_type`: `0` none,
`1` photo, `2` video.

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `access` | Campaign is not yours |
| `db` | Database error |

---

<a id="camp-broadcast-set"></a>

## Save one block

`POST /api/camp/broadcast.json`

Creates or updates the block for one status.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `status` | ✓ | Lead status `0`–`4` |
| `on` | | `1` enable, `0` disable |
| `delay` | | Seconds after entering the status before sending |
| `text` | | Message text (Telegram Markdown) |
| `media` | | Media link from [`media/upload`](campaigns.md#media-upload); `""` if none. Checked for ownership |
| `media_type` | | `1` photo, `2` video (from the upload response) |
| `button` | | Button caption |
| `link` | | Button URL (macros allowed); validated for correctness |

```json
{ "status": "ok" }
```

Enabling a block (`on=1`) requires the broadcasts feature → otherwise `error: broadcasts`.
Disabling or editing an already-disabled block is always allowed (so that after a
downgrade a block can still be turned off). Changing the media resets the cached `file_id`.

| Error | Meaning |
|---|---|
| `func` | `id`/`status` invalid, or the link is malformed, or the media is not yours |
| `broadcasts` | Enabling a block on a tariff without the broadcasts feature |
| `access` | Campaign is not yours |
| `db` | Database error |

---

<a id="camp-btest"></a>

## Send a block to myself

`POST /api/camp/btest.json`

Delivers a preview of one saved status block to the authorized user's Telegram via the
service bot (text and button only — media is not included).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `status` | ✓ | Lead status `0`–`4` |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id`/`status` invalid |
| `access` | Campaign is not yours |
| `db` | Database error |

---

<a id="broadcast-list"></a>

## List broadcasts

`GET /api/broadcast/list.json`

Returns the user's manual broadcasts, newest first, with the **stored** counters.
Broadcast statuses: `0` draft · `1` running · `2` done · `3` cancelled · `4` paused.

No parameters. Each card carries the full set of fields (the same as
[`get`](broadcasts.md#broadcast-get)), but without a live recompute: `remaining`
is always `0`, `total` is the stored value, and `campaigns` is `null` (the campaign list
is filled only by `get`).

```json
{
  "status": "ok",
  "data": [
    {
      "id": 5,
      "name": "May win-back",
      "status": 1,
      "statuses": 24,
      "campaigns": null,
      "created_from": 0, "created_to": 0,
      "status_from": 1714521600, "status_to": 1717113600,
      "text": "We miss you — here's 20% off 🎁",
      "media": "", "media_type": 0,
      "button": "Claim", "link": "https://example.com/back?click={click}",
      "total": 1240, "remaining": 0,
      "queued": 440, "sent": 800, "failed": 0,
      "started": 1717500000, "created": 1717490000
    }
  ]
}
```

`statuses` is a **bitmask** of the target lead statuses (bit 0 wait … bit 4 trash);
`24` = bits 3+4 = cancel + trash.

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="broadcast-quota"></a>

## Daily limit gauge

`GET /api/broadcast/quota.json`

The daily manual-send budget: the cap from the tariff, manual sends used today (the day
in UTC) and the current send-queue depth. No parameters.

```json
{ "status": "ok", "data": { "cap": 500, "used": 120, "queued": 440 } }
```

| Field | Meaning |
|---|---|
| `cap` | Daily manual-send cap from the tariff; `0` = unlimited |
| `used` | Manual messages sent today (the day in UTC); `0` if there is no row for the day yet |
| `queued` | All of the user's pending queued messages (triggered + manual) |

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="broadcast-get"></a>

## Get one broadcast

`GET /api/broadcast/get.json`

Returns the broadcast in full, including the campaign scope and filters. For a
**non-running** broadcast (not in the `running` status) it recomputes `total` (the live
audience) and `remaining` (the size of **Top up** — leads not yet reached). For a running
broadcast the stored counters are left as is.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{
  "status": "ok",
  "data": {
    "id": 5, "name": "May win-back", "status": 2, "statuses": 24,
    "campaigns": [12, 18],
    "created_from": 0, "created_to": 0,
    "status_from": 1714521600, "status_to": 1717113600,
    "text": "We miss you — here's 20% off 🎁",
    "media": "", "media_type": 0,
    "button": "Claim", "link": "https://example.com/back?click={click}",
    "total": 1300, "remaining": 60,
    "queued": 0, "sent": 1240, "failed": 0,
    "started": 1717500000, "created": 1717490000
  }
}
```

`campaigns` is normalized to `[]` (never `null`); an empty list = all your campaigns.

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `access` | Broadcast is not yours or not found |
| `db` | Database error |

---

<a id="broadcast-add"></a>

## Create a draft

`POST /api/broadcast/add.json`

Creates an empty broadcast draft. Requires the broadcasts feature. A per-user soft cap of
**100** broadcasts applies: on reaching it you get `error: limit` — delete an unneeded one
to free a slot.

| Parameter | Description |
|---|---|
| `name` | Optional display name |

```json
{ "status": "ok", "id": 5 }
```

`id` arrives as a **top-level field** next to `status`, not inside `data`.

| Error | Meaning |
|---|---|
| `broadcasts` | The broadcasts feature is not enabled on the tariff |
| `limit` | Already 100 broadcasts — delete one to create a new one |
| `db` | Database error |

---

<a id="broadcast-edit"></a>

## Edit a broadcast

`POST /api/broadcast/edit.json`

Replaces the broadcast's fields and campaign scope. Requires the broadcasts feature.
Allowed in **any** state — the body is taken at send time, so editing a running broadcast
affects messages that haven't gone out yet. The link is validated for correctness, and the
media for ownership; otherwise `error: func`.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |
| `name` | | Display name |
| `statuses` | | Bitmask of target statuses (bit 0 wait … bit 4 trash) |
| `campaigns` | | Comma-separated campaign IDs; empty = all your campaigns. Foreign IDs are silently dropped |
| `created_from` / `created_to` | | Filter by lead creation time (unix; `0` = no bound) |
| `status_from` / `status_to` | | Filter by lead status-change time (unix; `0` = no bound) |
| `text` | | Message text (Telegram Markdown) |
| `media` / `media_type` | | Media link from [`media/upload`](campaigns.md#media-upload) and its type (`1` photo, `2` video) |
| `button` / `link` | | Button caption + URL (macros allowed) |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set, or the link is malformed, or the media is not yours |
| `broadcasts` | The broadcasts feature is not enabled on the tariff |
| `access` | Broadcast is not yours |
| `db` | Database error |

---

<a id="broadcast-start"></a>

## Start (always "top up")

`POST /api/broadcast/start.json`

Queues the audience and flips the broadcast to **running**. Requires the broadcasts
feature. This is always a **top up**: only leads the broadcast hasn't reached yet get
queued, so re-starting a `done`/`cancelled` broadcast goes out only to new leads. Allowed
from `draft`/`done`/`cancelled`; a broadcast in the `running`/`paused` state returns
`error: state`.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{ "status": "ok", "queued": 60 }
```

`queued` arrives as a **top-level field** — the number of messages put in the queue. `0`
means there was nobody to reach, and the broadcast status does not change:

```json
{ "status": "ok", "queued": 0 }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `broadcasts` | The broadcasts feature is not enabled on the tariff |
| `state` | Broadcast is `running` or `paused` — use pause/cancel or resume |
| `access` | Broadcast is not yours |
| `db` | Database error |

---

<a id="broadcast-pause"></a>

## Pause

`POST /api/broadcast/pause.json`

`running → paused`: unsent messages are parked and preserved (the sender skips them but
does not lose them).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `state` | Broadcast is not `running` or not yours |
| `db` | Database error |

---

<a id="broadcast-resume"></a>

## Resume

`POST /api/broadcast/resume.json`

`paused → running`: the parked messages become current again and sending continues exactly
where it stopped.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `state` | Broadcast is not `paused` or not yours |
| `db` | Database error |

---

<a id="broadcast-reset"></a>

## Reset

`POST /api/broadcast/reset.json`

Clears the set of reached leads and the counters, and returns a
`draft`/`done`/`cancelled` broadcast to `draft`, keeping the name, filters and body. The
next start then re-sends to the **whole** audience. A broadcast in the `running`/`paused`
state must be cancelled first.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `state` | Broadcast is `running`/`paused` (cancel it first) or not yours |
| `db` | Database error |

---

<a id="broadcast-cancel"></a>

## Stop (cancel)

`POST /api/broadcast/cancel.json`

`running`/`paused → cancelled`: drops the remaining messages in the queue. The set of
already-reached leads is preserved, so a future **Top up** start won't write to them
again.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `state` | Broadcast is not `running`/`paused` or not yours |
| `db` | Database error |

---

<a id="broadcast-del"></a>

## Delete

`POST /api/broadcast/del.json`

Deletes the broadcast and its scope, queue and reach records. A `paused` broadcast **can**
be deleted (its parked messages go with it); only the `running` state is blocked — it must
be cancelled first.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `state` | Broadcast is `running` (cancel it first) or not yours |
| `db` | Database error |

---

<a id="broadcast-test"></a>

## Send to myself

`POST /api/broadcast/test.json`

Delivers a preview of the broadcast body to the authorized user via the service bot (text
and button only — media is not included). It does **not** require the broadcasts feature —
you can test an existing broadcast even on a tariff without it.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Broadcast ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` not set (`0`) |
| `access` | Broadcast is not yours |
| `db` | Database error |

---

<a id="errors"></a>

## Errors

| Error | Meaning |
|---|---|
| `broadcasts` | The broadcasts feature is not enabled on the tariff — [upgrade your tariff](https://docs.altercpa.top/en/guide/billing/) |
| `limit` | The 100-broadcasts-per-user limit has been reached |
| `state` | The action is not allowed for the broadcast's current status (e.g. starting a running one) |
| `access` | Broadcast or campaign is not yours / not found |
| `func` | A required parameter is missing or invalid (or the link is malformed / the media is not yours) |
| `unpaid` | Subscription is inactive |
| `db` | Database error |
