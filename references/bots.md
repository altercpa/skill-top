# Bots

Connecting and managing the Telegram bots that greet subscribers and track joins
to your chats, plus editing the bot profile (name, description, avatar).

Every endpoint in this section requires authentication with your API key
(see [Profile and API key](profile.md#profile-apikey)) **and an active paid
subscription**. Without a subscription the request returns the `unpaid` error; with a
missing or invalid key — `key`; for a banned account — `ban`.

Every response is HTTP 200 with `Content-Type: application/json`. Success or failure
is signalled by the `status` field in the body, not by the HTTP code. An error always
comes back as `{"status":"error","error":"<code>"}`.

---

<a id="bots-list"></a>

## Bots list

`GET /api/bots/list.json`

Returns the bots connected to your account, sorted newest to oldest. Soft-deleted bots
(status `3`) are excluded from the list. When the list is opened, bot names and
`@username`s are synced in the background with Telegram (at most one sweep per user
every 5 minutes), so a bot renamed in Telegram is picked up without a manual
reconnect.

```json
{
  "status": "ok",
  "data": [
    {
      "id": 7,
      "tg": 7123456789,
      "username": "mytrackbot",
      "name": "My Tracking Bot",
      "status": 0,
      "created": 1746100800,
      "default_campaign": 0
    }
  ]
}
```

`data` is an array (empty `[]` when there are no bots). The `created` field is Unix
time in seconds (an integer), not a date string. The `default_campaign` field is the
id of the [default campaign](bots.md#bots-default) launched when the bot is
opened without a campaign code; `0` means none is pinned (the newest active campaign is
used instead).

| `status` | Meaning |
|---|---|
| `0` | Active |
| `1` | Disabled |
| `2` | Token invalid |
| `4` | Sleep (paused because the subscription lapsed; resumes on renewal) |

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="bots-add"></a>

## Connect a bot

`POST /api/bots/add.json`

Connects a bot by its BotFather token. The token is validated via the Telegram API
(`getMe`), stored encrypted, and then a webhook is registered to receive updates.

| Parameter | Required | Description |
|---|---|---|
| `token` | ✓ | Bot token from @BotFather, e.g. `1234567890:AABBCCdd…` |

```json
{
  "status": "ok",
  "data": {
    "id": 7,
    "tg": 7123456789,
    "username": "mytrackbot",
    "name": "My Tracking Bot"
  }
}
```

The meaningful fields in the response are `id`, `tg`, `username` and `name`; the
`status` and `created` fields are not populated here.

**Recovery on re-adding:** if you previously disconnected this bot and add the same
token again, the existing record is reactivated (status becomes active) — no duplicate
is created, and the bot's campaigns are preserved. A token already linked to another
account is rejected.

| Error | Meaning |
|---|---|
| `token` | Token not provided, invalid or revoked (Telegram rejected `getMe`) |
| `exists` | This Telegram bot is already connected to another account |
| `webhook` | Token is valid, but webhook registration failed (the new record is rolled back) |
| `func` | Internal error (token encryption or client construction) |
| `db` | Database error |

---

<a id="bots-chats"></a>

## Chats where the bot is an administrator

`GET /api/bots/chats.json`

Returns the chats where the bot is an administrator (read from our backend's cache),
**excluding** chats already occupied by one of your live campaigns (each chat can be
used in only one campaign). It is used as the selection list when creating a campaign.
An empty list is normal (the bot was just added, isn't an admin anywhere, or all its
chats are already taken).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Bot ID |

```json
{
  "status": "ok",
  "data": [
    { "id": -1001234567890, "title": "My Channel", "type": "channel" }
  ]
}
```

`data` is an array (empty `[]` on a cache miss or if all chats are already taken). The
`type` field is the Telegram chat type string (`channel`, `group`, `supergroup`). The
cache is filled the moment the bot is granted administrator rights in a chat; if an
expected chat is missing, make sure the bot really is an administrator and retry the
request.

| Error | Meaning |
|---|---|
| `func` | Bot ID not provided or zero |
| `access` | The bot does not belong to your account |

---

<a id="bots-campaigns"></a>

## Bot campaigns

`GET /api/bots/campaigns.json`

Returns this bot's campaigns (excluding deleted ones, newest first) — the selection
list for pinning a [default campaign](bots.md#bots-default).

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Bot ID |

```json
{
  "status": "ok",
  "data": [
    { "id": 42, "name": "Summer promo", "status": 0 }
  ]
}
```

`data` is an array (empty `[]` if the bot has no campaigns). The `status` field matches
the campaign status (`0` active, `1` paused, etc.).

| Error | Meaning |
|---|---|
| `func` | Bot ID not provided or zero |
| `access` | The bot does not belong to your account |
| `db` | Database error |

---

<a id="bots-default"></a>

## Default campaign

`POST /api/bots/default.json`

Pins the campaign a bot runs when a subscriber opens it **without a campaign code** (a
plain `/start`). If no campaign is pinned, the bot's newest active campaign is used. The
pinned campaign must be one of this bot's live (non-deleted) campaigns.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Bot ID |
| `campaign` | ✓ | Campaign id to pin; `0` clears the pin (falls back to "newest active") |

```json
{ "status": "ok" }
```

The pin is advisory: if you later pause or delete the pinned campaign, the bot
gracefully falls back to the newest active one — there's no need to reset the field.

| Error | Meaning |
|---|---|
| `func` | Bot ID not provided or zero |
| `access` | The bot isn't yours, or the given campaign is not found / not on this bot / deleted |
| `db` | Database error |

---

<a id="bots-del"></a>

## Disconnect a bot

`POST /api/bots/del.json`

Soft-deletes a bot (status `3`), drops the Telegram webhook (best-effort) and clears
the cache. The record is retained for history; all leads, journal and statistics
referencing this bot remain. The bot can be reconnected — the "Recovery on re-adding"
logic applies.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Bot ID |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | Bot ID not provided or zero |
| `access` | Record not found or does not belong to your account |
| `db` | Database error |

---

<a id="bots-profile"></a>

## Bot profile

`GET /api/bots/profile.json`

Returns the bot's profile from the Telegram side (name, description and short
description), querying it directly from the Bot API (`getMyName`, `getMyDescription`,
`getMyShortDescription`). These fields are not stored in the database — Telegram is the
source of truth.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Bot ID |

```json
{
  "status": "ok",
  "data": {
    "name": "My Tracking Bot",
    "description": "This bot greets new subscribers and tracks joins.",
    "short_description": "Telegram join tracking"
  }
}
```

| Error | Meaning |
|---|---|
| `func` | Bot ID not provided/zero or internal error (token decryption) |
| `access` | Bot not found or does not belong to your account |
| `db` | Database error |
| `tg` | One of the Bot API requests failed |

---

<a id="bots-profile-set"></a>

## Save bot profile

`POST /api/bots/profile.json`

Saves the bot's name, description and short description by sending them to Telegram
(`setMyName`, `setMyDescription`, `setMyShortDescription`). The new name is also
mirrored into the bot record, so the [bots list](bots.md#bots-list) reflects the
change immediately.

| Parameter | Required | Description |
|---|---|---|
| `id` | ✓ | Bot ID |
| `name` | | Bot name. Sent to Telegram as-is (including an empty string) and mirrored to the bots list |
| `description` | | Full bot description |
| `short_description` | | Short description (the "about" text) |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | Bot ID not provided/zero or internal error (token decryption) |
| `access` | Bot not found or does not belong to your account |
| `db` | Database error |
| `tg` | One of the Bot API requests failed |

---

<a id="bots-photo"></a>

## Bot avatar

`POST /api/bots/photo.json`

Accepts an image as part of a multipart form (the `file` field) and sets it as the
bot's profile photo via Telegram (`setMyProfilePhoto`). The file is passed straight to
Telegram and stored nowhere. JPEG, PNG or WEBP up to 5 MiB are allowed. The request is
sent as `multipart/form-data`.

| Parameter | Required | Description |
|---|---|---|
| `file` | ✓ | Image file (JPEG/PNG/WEBP, up to 5 MiB) |
| `id` | ✓ | Bot ID (form field) |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `size` | Could not parse the form or the body size exceeds 5 MiB |
| `func` | Bot ID not provided/zero, the `file` part is missing, or an internal error |
| `access` | Bot not found or does not belong to your account |
| `db` | Database error |
| `tg` | The `setMyProfilePhoto` Bot API request failed |
