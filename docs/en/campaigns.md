# Campaigns, screens & media

Full campaign management (CRUD, deep link, pause/resume), tracker integration
(flow and its statistics), triggered broadcasts, multi-screen welcome flows with
buttons, CAS protection, group cleanup and media upload.

Every endpoint on this page requires an active subscription (see [Authentication & conventions](authentication.md#conventions)).
On an expired subscription the response is `unpaid`; on invalid authentication —
`key`. The response always comes back with an **HTTP 200** code and an
`application/json` body; success or failure is reported in the `status` field.
The common rules live in [Authentication & conventions](authentication.md#conventions).

---

<a id="camp-list"></a>

## List campaigns

`GET /api/camp/list.json`

Returns all non-deleted campaigns (the campaign status ≠ 2) of the current account,
newest first. Each campaign is a full card with lifetime counters and (if a flow
has been created) the resolved tracker flow URL.

```json
{
  "status": "ok",
  "data": [
    {
      "id": 12,
      "code": "a1b2c3d4",
      "bot": 7,
      "bot_username": "mytrackbot",
      "chat": -1001234567890,
      "chat_type": 0,
      "chat_title": "My Channel",
      "chat_about": "Exclusive content for subscribers",
      "chat_link": "https://t.me/mychannel",
      "name": "Летняя кампания",
      "text": "Подпишитесь на *{title}* — эксклюзив внутри!",
      "media": "",
      "media_type": 0,
      "button": "Получить доступ",
      "link": "https://example.com/lander?click={click}",
      "mode": 0,
      "cas_ban": 0,
      "auto_accept": 0,
      "pb_wait": "",
      "pb_hold": "",
      "pb_approve": "",
      "pb_cancel": "",
      "pb_trash": "",
      "unsub_window": 2592000,
      "track_organic": 1,
      "notify_level": 1,
      "flow_id": 0,
      "flow_url": "",
      "status": 0,
      "created": 1746860400,
      "counters": {
        "wait": 120,
        "hold": 30,
        "approve": 98,
        "cancel": 8,
        "trash": 4,
        "org_join": 15,
        "org_leave": 3
      }
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `code` | 8-character campaign code, used in the deep link |
| `chat_type` | `0` channel, `1` group |
| `chat_about` | Channel/group description pulled from Telegram (read-only) |
| `chat_link` | Public chat link (`t.me/<username>` or invite), `""` if unknown (read-only) |
| `mode` | Welcome type: `0` classic single message, `1` multi-screen flow |
| `cas_ban` | `1` — kick joiners listed in CAS (see [CAS protection](#camp-cas)) |
| `auto_accept` | `1` — automatically approve incoming join requests (see [Create campaign](#camp-add)) |
| `track_organic` | `1` — log organic joins/leaves, `0` — don't |
| `notify_level` | Lead notifications: `0` off, `1` approve, `2` approve + wait, `3` all statuses |
| `flow_id` / `flow_url` | Tracker flow bound to the campaign (`0`/`""` until a flow is created) |
| `created` | Unix creation time (integer seconds) |
| `counters` | Lifetime counters from the daily rollups; zeros until there is stats data |

| `status` | Meaning |
|---|---|
| `0` | Active |
| `1` | Paused |
| `2` | Deleted (soft — leads and stats are kept; not shown in the list) |
| `3` | Sleeping (suspended by the worker on tariff expiry; woken on renewal) |

---

<a id="camp-get"></a>

## Get one campaign

`GET /api/camp/get.json`

Returns the card of a single campaign by `id` (same structure as in the list) with
lifetime counters and the resolved flow URL.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{
  "status": "ok",
  "data": {
    "id": 12,
    "code": "a1b2c3d4",
    "bot": 7,
    "bot_username": "mytrackbot",
    "chat": -1001234567890,
    "chat_type": 0,
    "chat_title": "My Channel",
    "chat_about": "Exclusive content for subscribers",
    "chat_link": "https://t.me/mychannel",
    "name": "Летняя кампания",
    "text": "Подпишитесь на *{title}* — эксклюзив внутри!",
    "media": "",
    "media_type": 0,
    "button": "Получить доступ",
    "link": "https://example.com/lander?click={click}",
    "mode": 0,
    "cas_ban": 0,
    "auto_accept": 0,
    "pb_wait": "",
    "pb_hold": "",
    "pb_approve": "",
    "pb_cancel": "",
    "pb_trash": "",
    "unsub_window": 2592000,
    "track_organic": 1,
    "notify_level": 1,
    "flow_id": 0,
    "flow_url": "",
    "status": 0,
    "created": 1746860400,
    "counters": {
      "wait": 120,
      "hold": 30,
      "approve": 98,
      "cancel": 8,
      "trash": 4,
      "org_join": 15,
      "org_leave": 3
    }
  }
}
```

Empty `pb_*` fields mean the campaign inherits the
[global postback templates](profile.md#profile-postbacks).
`unsub_window` is in seconds; `0` means the window never closes (cancel/trash
postbacks are always sent, regardless of the unsubscribe time).

| Error | Meaning |
|---|---|
| `func` | `id` not supplied or equal to `0` |
| `access` | Campaign not found or belongs to another account |

---

<a id="camp-add"></a>

## Create campaign

`POST /api/camp/add.json`

Creates a campaign on a bot and chat you own and that are not deleted. Your plan's
campaign limit applies (`-1` = unlimited). The name
and chat type are pulled from the bot's admin-chats cache; an 8-character code is
generated automatically.

`bot` and `chat` must be taken from [`bots/chats`](bots.md#bots-chats) — this
is an admin bot and a chat it belongs to.

| Parameter | Required | Description |
|---|---|---|
| `bot` | ✓ | Bot ID |
| `chat` | ✓ | Chat ID (signed number, from [`bots/chats`](bots.md#bots-chats)) |
| `name` | | Campaign name |
| `text` | | Welcome message text (Telegram Markdown) |
| `media` | | Media ref from [`media/upload`](#media-upload); omit for a text message |
| `media_type` | | Media type: `1` photo, `2` video |
| `button` | | Label of the CPA button in the welcome message |
| `link` | | CPA button URL — may contain `{click}` and other macros (validated: `https://`, `http://`, `tg://`, `t.me/`, `telegram.me/`, `@handle` or empty) |
| `pb_wait` | | Postback URL template for the **wait** event |
| `pb_hold` | | Postback URL template for the **hold** event |
| `pb_approve` | | Postback URL template for the **approve** event |
| `pb_cancel` | | Postback URL template for the **cancel** event |
| `pb_trash` | | Postback URL template for the **trash** event |
| `unsub_window` | | Seconds after the join during which cancel/trash postbacks are sent; `0` = always |
| `track_organic` | | Log organic joins/leaves. **Enabled** by default; turned off only by an explicit falsy value (`0`/`false`/`no`/`off`) |
| `auto_accept` | | Automatically approve incoming chat join requests. **Disabled** by default; turned on by an explicit truthy value (`1`/`true`/`yes`/`on`). Not tariff-gated |
| `notify_level` | | Lead notification level: `0` off, `1` approve, `2` approve + wait, `3` all statuses |

```json
{ "status": "ok", "id": 12, "code": "a1b2c3d4" }
```

On creation the chat's title, description and public link are pulled from Telegram
(non-critical — backfilled later if unavailable) and stored on the card
(`chat_title`, `chat_about`, `chat_link`).

| Error | Meaning |
|---|---|
| `func` | `bot` or `chat` equal `0`, or `link` failed validation |
| `access` | The bot isn't yours or is deleted |
| `limit` | The per-tariff campaign limit was reached — [upgrade your tariff](https://docs.altercpa.top/en/guide/billing/) |

---

<a id="camp-edit"></a>

## Edit campaign

`POST /api/camp/edit.json`

Partial update: only the supplied fields change, the rest stay as they are. Each
field, including each of the `pb_*`, is edited independently. The bot and chat
cannot be changed after creation. Changing `media` resets the cached Telegram
`file_id` (the file is re-uploaded on the next `/start`).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `name` | | New campaign name |
| `text` | | New welcome message text |
| `media` | | New media ref (must belong to you; pass `""` to remove the current media) |
| `media_type` | | New media type: `1` photo, `2` video |
| `button` | | New CPA button label |
| `link` | | New CPA button URL |
| `pb_wait` | | New **wait** postback URL template |
| `pb_hold` | | New **hold** postback URL template |
| `pb_approve` | | New **approve** postback URL template |
| `pb_cancel` | | New **cancel** postback URL template |
| `pb_trash` | | New **trash** postback URL template |
| `unsub_window` | | New unsubscribe window in seconds |
| `track_organic` | | `1`/`true`/`yes`/`on` — log organic, otherwise don't |
| `auto_accept` | | `1`/`true`/`yes`/`on` — auto-approve join requests, otherwise don't |
| `notify_level` | | New lead notification level (0–3) |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0`, or `link` or your own `media` failed validation |
| `access` | The campaign isn't yours |

---

<a id="camp-link"></a>

## Get deep link

`GET /api/camp/link.json`

Returns the subscriber deep link for this campaign and a tracker-connection flag.
Distribute this link through your traffic sources — when a subscriber opens it, the
bot greets them and the conversion is recorded.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{
  "status": "ok",
  "code": "a1b2c3d4",
  "url": "https://t.me/mytrackbot?start=a1b2c3d4-{click}",
  "tracker_connected": true
}
```

The link always contains the literal `{click}` macro — the advertiser's tracker
substitutes the real click ID at the moment of the redirect. `tracker_connected`
indicates whether you have a tracker connected (whether a flow can be created via
[`camp/flow`](#camp-flow)).

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours |

---

<a id="camp-pause"></a>

## Pause or resume

`POST /api/camp/pause.json`

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `pause` | | `1` — pause, `0`/absent — resume |

```json
{ "status": "ok", "camp_status": 1 }
```

`camp_status` is the campaign's new status in the response: `1` paused, `0` active.

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours |

---

<a id="camp-del"></a>

## Delete campaign

`POST /api/camp/del.json`

Soft-deletes the campaign (the campaign status = 2). Existing leads, journal records and
stats are kept and continue to reference the campaign.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours |

---

<a id="camp-mode"></a>

## Switch welcome mode

`POST /api/camp/mode.json`

Switches the welcome flow between a classic single message (`mode` = 0) and a
multi-screen flow (`mode` = 1). The first time screen mode is turned on, an entry
screen is seeded from the classic creative (text/media + CTA button), so the switch
loses no data and is immediately usable.

Turning on screen mode requires the Screens feature on your tariff (checked on the
server); turning it off is always allowed — a user without the feature can return to
the classic message.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `mode` | ✓ | `1` — multi-screen flow, `0` — classic message (field must be present) |

```json
{ "status": "ok", "mode": 1 }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0`, or `mode` not supplied |
| `access` | The campaign isn't yours |
| `screens` | The screens feature isn't enabled on the tariff (only when turning on) — [upgrade your tariff](https://docs.altercpa.top/en/guide/billing/) |

---

<a id="camp-flow"></a>

## Create tracker flow

`POST /api/camp/flow.json`

Creates a flow in the connected AlterCPA Lite tracker for the campaign's deep link,
saves the flow ID and URL onto the campaign, and returns the URL. If a flow already
exists, it returns the stored URL without contacting the tracker. Requires a
connected tracker (see [connecting a tracker](profile.md#profile-tracker-set)).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{ "status": "ok", "url": "https://track.example.com/abcd1234" }
```

If the flow-create call to the tracker fails, the response stays `status: ok` but
carries a soft error:

```json
{ "status": "ok", "error": "create" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours, or no tracker is connected |
| `create` | The tracker couldn't create the flow (soft error, `status` = `ok`) |

---

<a id="camp-flowsync"></a>

## Re-sync tracker flow

`POST /api/camp/flowsync.json`

Restores a campaign's link to its tracker flow if the flow may have been deleted on
the tracker side. It probes the saved flow: if it's still there, the URL is refreshed
and `recreated: false` is returned; if the tracker reports the flow is gone, it is
recreated from the campaign's deep link, the new ID and URL are saved, and
`recreated: true` is returned. If the tracker is unreachable, the flow is **not**
recreated (to avoid duplicates). Use this call to recover a flow deleted on the
tracker side.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{ "status": "ok", "url": "https://track.example.com/abcd1234", "recreated": false }
```

Soft errors come back with `status: ok` and an `error` field:

```json
{ "status": "ok", "error": "sync" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours, or no tracker is connected |
| `noflow` | The campaign has no flow yet — create one via [`camp/flow`](#camp-flow) first (soft error, `status` = `ok`) |
| `sync` | The tracker is unreachable, recreation skipped (soft error, `status` = `ok`) |
| `create` | The flow was deleted on the tracker but recreation failed (soft error, `status` = `ok`) |
| `busy` | Too many requests (limit 30 per minute) |

---

<a id="camp-flowstats"></a>

## Flow statistics

`GET /api/camp/flowstats.json`

Returns daily statistics from the connected tracker for the campaign's flow over the
last N days.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `days` | | Number of days; default `7`, max `90` |

```json
{
  "status": "ok",
  "rows": [
    {
      "date": "2026-06-20",
      "clicks": 540,
      "unique": 512,
      "wait": 120,
      "hold": 30,
      "approve": 98,
      "cancel": 8,
      "trash": 4,
      "cr": 19.14,
      "ar": 81.67,
      "exit_pct": 10.91
    }
  ]
}
```

If the campaign has no flow yet, `rows` is empty (`[]`). If the request to the
tracker fails, the response stays `status: ok` with a soft error
`{"status":"ok","error":"stats"}`.

| Field | Meaning |
|---|---|
| `clicks` / `unique` | Total clicks / unique clicks |
| `cr` | CR%: valid leads / unique clicks (computed by the tracker) |
| `ar` | AR%: approved / valid leads (computed by the tracker) |
| `exit_pct` | (cancel + trash) / (approve + cancel + trash), % |

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours, or no tracker is connected |
| `stats` | The tracker couldn't return statistics (soft error, `status` = `ok`) |

---

<a id="camp-broadcast"></a>

## Triggered broadcasts: get

`GET /api/camp/broadcast.json`

Returns the campaign's five triggered auto-message blocks — one per lead status
(`0` wait, `1` hold, `2` approve, `3` cancel, `4` trash). A status with no saved
block is returned as the zero block (`on` = 0). These are the **triggered**
per-status broadcasts; bulk manual broadcasts are described on the
[Broadcasts](broadcasts.md) page.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{
  "status": "ok",
  "data": [
    {
      "status": 0,
      "on": 1,
      "delay": 3600,
      "text": "Ещё думаете? Доступ закрывается!",
      "media": "",
      "media_type": 0,
      "button": "Вступить",
      "link": "https://t.me/mytrackbot?start=a1b2c3d4-{click}"
    }
  ]
}
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours |

---

<a id="camp-broadcast-set"></a>

## Triggered broadcasts: save

`POST /api/camp/broadcast.json`

Saves (upserts) one broadcast block by status. Enabling (`on` = 1) requires the
Broadcasts feature on your tariff; disabling and editing a
disabled block are always allowed. Changing `media` resets the cached `file_id`.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `status` | ✓ | Lead status `0`–`4` (`>4` → `func`) |
| `on` | | `1` — enable the broadcast (requires the broadcasts feature), otherwise disable |
| `delay` | | Seconds after entering the status before sending |
| `text` | | Message text |
| `media` | | Media ref from [`media/upload`](#media-upload) (must belong to you) |
| `media_type` | | Media type: `1` photo, `2` video |
| `button` | | CTA button label |
| `link` | | CTA button URL (validated as for the campaign) |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0`, `status` > 4, or `link`/`media` failed validation |
| `access` | The campaign isn't yours |
| `broadcasts` | The broadcasts feature isn't enabled on the tariff (only when enabling) — [upgrade your tariff](https://docs.altercpa.top/en/guide/billing/) |

---

<a id="camp-btest"></a>

## Test broadcast

`POST /api/camp/btest.json`

Enqueues a "send to myself" — a preview of the broadcast block (text + CTA only) to
the owner via the service bot. The block must be saved beforehand.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `status` | ✓ | Block status `0`–`4` |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` or `status` > 4 |
| `access` | The campaign isn't yours |

---

<a id="screen-list"></a>

## Welcome screens: list

`GET /api/screen/list.json`

Returns the campaign's screens (entry screen first; sorted by `scr_order`, `scr_id`),
each with its ordered buttons. Owner access only; **not** restricted by the screens
feature (but requires an active subscription).

The multi-screen flow is used by the campaign when the campaign mode `mode` = 1 (see
[welcome mode](#camp-mode)). A screen is media + text + an ordered list of inline
buttons that the bot lays out in rows of `columns` width.

| Parameter | Required | Description |
|---|---|---|
| `camp` | ✓ | Campaign ID |

```json
{
  "status": "ok",
  "data": [
    {
      "id": 30,
      "campaign": 12,
      "order": 0,
      "name": "Старт",
      "text": "Привет! Готовы вступить?",
      "media": "",
      "media_type": 0,
      "columns": 1,
      "created": 1746860400,
      "buttons": [
        { "id": 51, "order": 0, "label": "Подробнее", "kind": 1, "link": "", "target": 31 },
        { "id": 52, "order": 1, "label": "Открыть сайт", "kind": 0, "link": "https://example.com/lander", "target": 0 }
      ]
    }
  ]
}
```

| Button field | Meaning |
|---|---|
| `kind` | `0` URL button (`link` set), `1` navigation to another screen (`target` set) |
| `target` | `scr_id` of the destination screen in the same campaign (for `kind` = 1) |

| Error | Meaning |
|---|---|
| `func` | `camp` equals `0` |
| `access` | The campaign isn't yours |

---

<a id="screen-add"></a>

## Add screen

`POST /api/screen/add.json`

Adds a screen to the campaign (limit `screens_max`, default 10). The first screen
gets `order` 0 (entry). Returns the new screen card.

| Parameter | Required | Description |
|---|---|---|
| `camp` | ✓ | Campaign ID |
| `name` | | Screen name |
| `text` | | Screen text |
| `media` | | Media ref from [`media/upload`](#media-upload) (must belong to you; empty is allowed) |
| `media_type` | | Media type: `1` photo, `2` video |
| `columns` | | Buttons per row, clamped to 1–8 (default 1) |

```json
{
  "status": "ok",
  "data": {
    "id": 32,
    "campaign": 12,
    "order": 2,
    "name": "Бонус",
    "text": "Дарим бонус новым подписчикам",
    "media": "",
    "media_type": 0,
    "columns": 1,
    "buttons": [],
    "created": 0
  }
}
```

| Error | Meaning |
|---|---|
| `func` | `camp` equals `0` or `media` failed validation |
| `access` | The campaign isn't yours |
| `screens` | The screens feature isn't enabled on the tariff — [upgrade your tariff](https://docs.altercpa.top/en/guide/billing/) |
| `limit` | The screen limit (`screens_max`) was reached |

---

<a id="screen-edit"></a>

## Edit screen

`POST /api/screen/edit.json`

Partial update of the screen's content: only the supplied fields change. Changing
`media` resets the cached `file_id`. Requires the screens feature on the tariff.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Screen ID |
| `name` | | New screen name |
| `text` | | New screen text |
| `media` | | New media ref (must belong to you) |
| `media_type` | | New media type: `1` photo, `2` video |
| `columns` | | Buttons per row, clamped to 1–8 |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` or `media` failed validation |
| `access` | The screen isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |

---

<a id="screen-copy"></a>

## Copy screen

`POST /api/screen/copy.json`

Duplicates a screen (text + media + buttons) and appends the copy to the same
campaign (limit `screens_max`). Navigation buttons keep their `target`. Requires the
screens feature.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Source screen ID |

```json
{
  "status": "ok",
  "data": {
    "id": 33,
    "campaign": 12,
    "order": 3,
    "name": "Старт",
    "text": "Привет! Готовы вступить?",
    "media": "",
    "media_type": 0,
    "columns": 1,
    "buttons": [
      { "id": 60, "order": 0, "label": "Открыть сайт", "kind": 0, "link": "https://example.com/lander", "target": 0 }
    ],
    "created": 1750000000
  }
}
```

If the copy can't be re-read from the database, the response is a fallback
`{"status":"ok","id":33}` (only `id`, no `data`).

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The screen isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |
| `limit` | The screen limit (`screens_max`) was reached |

---

<a id="screen-del"></a>

## Delete screen

`POST /api/screen/del.json`

Deletes a screen and its buttons. Refuses if it's the campaign's only screen or if a
navigation button still points to it. Requires the screens feature.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Screen ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The screen isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |
| `last` | This is the campaign's only screen — it can't be deleted |
| `linked` | A navigation button still points to this screen — redirect it first |

---

<a id="screen-reorder"></a>

## Reorder screens

`POST /api/screen/reorder.json`

Reassigns the screen order from a list of their IDs. The screen at position 0 becomes
the entry screen. IDs that don't belong to the campaign are silently skipped. Requires
the screens feature.

| Parameter | Required | Description |
|---|---|---|
| `camp` | ✓ | Campaign ID |
| `order` | ✓ | Comma-separated screen IDs in the desired order, e.g. `5,3,7` |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `camp` equals `0` or the `order` list is empty |
| `access` | The campaign isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |

---

<a id="screen-test"></a>

## Test screen

`POST /api/screen/test.json`

Sends a "send to myself" preview of the screen (text + buttons only; media is omitted)
to the owner via the service bot. Owner access only; **not** restricted by the screens
feature.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Screen ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The screen isn't yours |

---

<a id="button-add"></a>

## Add button

`POST /api/button/add.json`

Adds a button to a screen (limit `screen_buttons_max`, default 8). `kind`, `link` and
`target` are validated together. Requires the screens feature. Returns the new button
card.

| Parameter | Required | Description |
|---|---|---|
| `screen` | ✓ | Screen ID |
| `label` | | Button label |
| `kind` | | `0` URL button (default), `1` navigation to another screen |
| `link` | | Required when `kind` = 0: a valid URL (`https`/`http`/`tg`/`t.me`/`telegram.me`/`@handle`); forcibly cleared when `kind` = 1 |
| `target` | | Required when `kind` = 1: `scr_id` of another screen in the same campaign (not this screen); forcibly `0` when `kind` = 0 |

```json
{
  "status": "ok",
  "data": { "id": 61, "order": 1, "label": "Подробнее", "kind": 1, "link": "", "target": 31 }
}
```

| Error | Meaning |
|---|---|
| `func` | `screen` equals `0` or an invalid `kind`/`link`/`target` combination |
| `access` | The screen isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |
| `limit` | The button limit (`screen_buttons_max`) was reached |

---

<a id="button-edit"></a>

## Edit button

`POST /api/button/edit.json`

Fully replaces the button's `label`/`kind`/`link`/`target` (not a partial update —
all fields are rewritten as a unit). `kind`/`link`/`target` are validated together,
as on add. Requires the screens feature.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Button ID |
| `label` | | New button label |
| `kind` | | `0` URL button, `1` navigation |
| `link` | | Required when `kind` = 0 (a valid URL); cleared when `kind` = 1 |
| `target` | | Required when `kind` = 1 (another screen in the same campaign); when `kind` = 0 → `0` |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` or an invalid `kind`/`link`/`target` combination |
| `access` | The button isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |

---

<a id="button-del"></a>

## Delete button

`POST /api/button/del.json`

Deletes a button. Requires the screens feature.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Button ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The button isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |

---

<a id="button-reorder"></a>

## Reorder buttons

`POST /api/button/reorder.json`

Reassigns the order of one screen's buttons from a list of their IDs. IDs that don't
belong to the screen are silently skipped. Requires the screens feature.

| Parameter | Required | Description |
|---|---|---|
| `screen` | ✓ | Screen ID |
| `order` | ✓ | Comma-separated button IDs in the desired order |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `screen` equals `0` or the `order` list is empty |
| `access` | The screen isn't yours |
| `screens` | The screens feature isn't enabled on the tariff |

---

<a id="camp-cas"></a>

## CAS protection

`POST /api/camp/cas.json`

Turns the campaign's CAS protection on or off — auto-kicking joiners listed in the CAS
ban list. Turning it on requires the Screens feature on your tariff (the same feature
as for screens); turning it off is always allowed.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |
| `cas_ban` | ✓ | `1` — enable protection, `0` — disable (field must be present) |

```json
{ "status": "ok", "cas_ban": 1 }
```

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` or `cas_ban` not supplied |
| `access` | The campaign isn't yours |
| `screens` | The screens feature isn't enabled on the tariff (only when enabling) — [upgrade your tariff](https://docs.altercpa.top/en/guide/billing/) |

---

<a id="camp-clean"></a>

## Start group cleanup

`POST /api/camp/clean.json`

Starts a group cleanup: it gathers the observed member base (joined leads still in the
chat ∪ members from the journal marked as present) into a queue and arms a job that our
backend drains. Only one cleanup per campaign can run at a time. Requires the
Screens feature on your tariff.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

```json
{ "status": "ok", "clean_id": 5, "total": 842, "state": 0 }
```

If there is nobody to check, the job finishes immediately:
`{"status":"ok","clean_id":5,"total":0,"state":1}`.

| `state` | Meaning |
|---|---|
| `0` | Running |
| `1` | Finished |

| Error | Meaning |
|---|---|
| `screens` | The screens feature isn't enabled on the tariff — [upgrade your tariff](https://docs.altercpa.top/en/guide/billing/) |
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours |
| `running` | A cleanup of this campaign is already running |

---

<a id="camp-clean-status"></a>

## Cleanup status

`GET /api/camp/clean.json`

Returns the campaign's latest cleanup job for progress polling. Reading the status is
**not** restricted by the screens feature (unlike starting a cleanup).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Campaign ID |

If a cleanup has never been run:

```json
{ "status": "ok", "exists": false }
```

If a job exists:

```json
{
  "status": "ok",
  "exists": true,
  "state": 0,
  "total": 842,
  "checked": 310,
  "kicked_cas": 4,
  "kicked_dead": 12,
  "failed": 1,
  "started": 1750000000,
  "finished": 0
}
```

| Field | Meaning |
|---|---|
| `state` | `0` running, `1` finished |
| `total` / `checked` | Total members in the base / already checked |
| `kicked_cas` | Kicked per the CAS list |
| `kicked_dead` | Kicked as deleted/inactive accounts |
| `failed` | Couldn't be checked |
| `started` / `finished` | Unix start and finish time (`0` for an unfinished job) |

| Error | Meaning |
|---|---|
| `func` | `id` equals `0` |
| `access` | The campaign isn't yours |

---

<a id="media-upload"></a>

## Upload media

`POST /api/media/upload.json`

Uploads an image or video for a campaign welcome message, a screen, a broadcast or a
mailing. Returns the media ref and type code to save via `camp/add`, `camp/edit`,
`screen/*` and so on.

Send it as `multipart/form-data`, with the file in the `file` field. Accepted formats:
`jpg`, `jpeg`, `png`, `webp`, `gif` (photo) and `mp4`, `mov`, `webm` (video);
documents are not accepted. The size is limited (default ~20 MiB). The limit is 20
uploads per hour per account.

```json
{ "status": "ok", "media": "17/abc123.jpg", "media_type": 1 }
```

`media` is the path relative to the media directory (this is what you save onto the
campaign/screen). `media_type` is the type code: `1` photo, `2` video; pass it together
with `media`.

| Error | Meaning |
|---|---|
| `func` | No file supplied (no `file` part) or a save error |
| `type` | Invalid file format (not in the photo/video list) |
| `size` | The file exceeds the size limit |
| `busy` | The limit of 20 uploads per hour was exceeded |
