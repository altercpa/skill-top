# Transfer

Hand over a bot (with all of its campaigns) or a single campaign to another
account — in one atomic operation. See also: [Transfer guide](https://docs.altercpa.top/en/guide/transfer/).

All requests are authenticated (your API key). The `token`
and `resolve` endpoints work on any plan; `bot` and `camp` require the sender
themselves to have an active paid subscription. Every response is HTTP 200 with an
`application/json` body; success or failure is determined by the `status` field,
not by the HTTP code.

---

<a id="how-it-works"></a>

## How the transfer works

The transfer uses a **transfer token** — a 32-character alphanumeric code held by
each user, separate from the API key. It identifies the recipient without revealing
their Telegram ID or username. The sender obtains the recipient's transfer token
out of band (for example, via direct message), checks it with
[resolve](transfer.md#transfer-resolve) to confirm the transfer is possible,
and then calls the transfer endpoint.

**Acceptance conditions** are checked before any data is moved:

- The recipient must have a **paid** subscription on record — a free / never-paid account cannot receive bots or campaigns (it is rejected with `unpaid`).
- The recipient must have **enough free campaign slots** for the campaigns being
  transferred.

In addition, the **sender** themselves must be on an active paid subscription: the
`bot` and `camp` endpoints are behind a paid gate and will return an `unpaid` error
before they run if the sender's subscription is inactive.

**The fate of leads** is controlled by the `leads` parameter, which is **optional**
and defaults to `0`:

- `leads=1` (any non-zero value) — all leads, statistics, and history are
  re-attributed to the recipient.
- `leads=0` (the default when the parameter is omitted) — all leads of the
  transferred campaigns and their related records are **permanently deleted**. To
  keep the leads, you must pass `leads=1` explicitly.

All write operations run inside a single database transaction.

**CRM on transfer.** If the transferred bot has CRM enabled, its dialogs and custom
events follow the same `leads` choice (moved to the recipient or deleted) and are
**detached from your operators**; the bot is removed from your teams (your operators
lose access at once) and CRM mode is switched off on it — the recipient re-enables it.
When transferring a **single campaign**, dialogs stay with the previous bot (which
stays with you) — only the lead-tied events move.

---

<a id="transfer-token"></a>

## Get your transfer token

`GET /api/transfer/token.json`

Returns your current transfer token. If a token has not been generated yet, it is
created and stored on first access. Share it with whoever wants to hand a bot or a
campaign over to you.

```json
{ "status": "ok", "token": "Hk3mP9qL2xVnT7wRcZ8aBdFgJ4sYuE6H" }
```

| Error | Meaning |
|---|---|
| `db` | Database error while reading or saving the token |

---

<a id="transfer-token-regen"></a>

## Regenerate the transfer token

`POST /api/transfer/token.json`

Generates a new 32-character transfer token. The old token stops working
immediately — any transfer a sender initiated with the old token will fail with a
`notfound` error.

```json
{ "status": "ok", "token": "Qz5tNb1cW8yMr3kHdLpX7vJ2aFgUeS9o" }
```

| Error | Meaning |
|---|---|
| `db` | Database error while updating the token |

---

<a id="transfer-resolve"></a>

## Resolve the recipient

`GET /api/transfer/resolve.json`

Looks up an account name by its transfer token and checks acceptance readiness.
Call it before [transfer/bot](transfer.md#transfer-bot) or
[transfer/camp](transfer.md#transfer-camp) to make sure the transfer is
possible.

| Parameter | Required | Description |
|---|---|---|
| `token` | ✓ | The recipient's transfer token |

```json
{
  "status": "ok",
  "name": "Ivan Petrov",
  "paid": true,
  "slots": 12
}
```

`name` is the recipient account's display name. `paid` reflects whether the
recipient has a paid subscription on record — it is
**not** a live active-window check, so a once-paid account whose window has since
lapsed still reports `true`; it is `false` only for a free / never-paid account.
`slots` is the number of free campaign slots:

| `slots` value | Meaning |
|---|---|
| `-1` | Unlimited plan (recipient has a paid subscription on record) |
| `0` | No free slots: either a free / never-paid account (`paid: false`), or a limited plan that is already full |
| `> 0` | Plan limit minus the recipient's number of non-deleted campaigns |

| Error | Meaning |
|---|---|
| `func` | The `token` parameter was not passed or is empty |
| `notfound` | No account found with this transfer token |
| `db` | Database error during the lookup |

---

<a id="transfer-bot"></a>

## Transfer a bot

`POST /api/transfer/bot.json`

Transfers a bot and **all** of its non-deleted campaigns to the recipient. Each
campaign's leads are re-attributed or deleted depending on `leads`, and the journal
history follows the bot to its new owner.

| Parameter | Required | Description |
|---|---|---|
| `bot` | ✓ | Bot ID (you must own it) |
| `token` | ✓ | The recipient's transfer token |
| `leads` | | `1` — re-attribute leads to the recipient; `0` (default) — delete them permanently |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `bot` or `token` not passed |
| `self` | This is your own transfer token — you can't transfer to yourself |
| `access` | You are not the owner of this bot |
| `notfound` | The transfer token does not match any account |
| `unpaid` | The sender's subscription is inactive, or the recipient is a free / never-paid account |
| `limit` | The recipient does not have enough free slots for the transferred campaigns |
| `held` | The bot or any of its campaigns is frozen by moderation |
| `db` | Database or transaction error |

---

<a id="transfer-camp"></a>

## Transfer a campaign

`POST /api/transfer/camp.json`

Transfers a single campaign to the recipient. The recipient must have an active bot
that is an administrator of the campaign's chat — a suitable bot is picked
automatically and becomes the campaign's new bot.

If the recipient's selected bot differs from the previous bot, the welcome media is
**re-uploaded** to the new bot (a Telegram file_id is bound to a bot): the media is
briefly sent to the campaign's chat and immediately deleted — a short message flash
may appear in the channel. If the re-upload fails, it does not abort the transfer:
the media is cleared, and the new owner will need to upload it again.

| Parameter | Required | Description |
|---|---|---|
| `camp` | ✓ | Campaign ID (you must own it) |
| `token` | ✓ | The recipient's transfer token |
| `leads` | | `1` — re-attribute leads (and rebind them to the new bot); `0` (default) — delete them permanently |

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `func` | `camp` or `token` not passed |
| `self` | This is your own transfer token — you can't transfer to yourself |
| `access` | You are not the owner of this campaign |
| `notfound` | The transfer token does not match any account |
| `unpaid` | The sender's subscription is inactive, or the recipient is a free / never-paid account |
| `limit` | The recipient has no free slot for the campaign |
| `nobot` | The recipient has no active bot that is an administrator of this campaign's chat |
| `held` | The campaign is frozen by moderation |
| `db` | Database or transaction error |

---

<a id="transfer-self"></a>

## Move a campaign between your own bots

Re-points a campaign to **another of your own bots** (same owner) — for example, if
the previous bot stopped working (lost admin rights or its token was revoked) and
another of your bots already administers the same group/channel. The recipient
doesn't change, so no token and no payment/slot checks are needed, and **leads are
always kept** (the `leads` parameter is not used here).

In one transaction the following move to the new bot: the campaign itself, the chat's
organic journal history, the CRM dialogs of the campaign's subscribers (for operator
chat to keep working, the new bot must be in the operator's team) and any in-flight
group cleanup. The welcome media is re-uploaded via the **previous** bot (this works
even if it merely lost group admin); if the previous bot is unreachable the move still
succeeds and the media is cleared (`media_lost: true` — re-upload it). A campaign (or
its bot) frozen by moderation is not eligible for the move.

---

<a id="transfer-targets"></a>

## Bots available for the move

`GET /api/transfer/targets.json`

Returns your **other** active bots that administer the campaign's chat — the valid
destinations for the move.

| Parameter | Required | Description |
|---|---|---|
| `camp` | ✓ | Campaign ID (you must own it) |

```json
{
  "status": "ok",
  "data": [
    { "id": 9, "name": "Backup Bot", "username": "mybackupbot" }
  ]
}
```

`data` is an array (may be empty if there are no suitable bots).

| Error | Meaning |
|---|---|
| `func` | `camp` equals `0` |
| `access` | The campaign isn't yours |
| `db` | Database error |

---

<a id="transfer-move"></a>

## Move a campaign to your bot

`POST /api/transfer/move.json`

Re-points the campaign to the specified bot of yours (one of the
[available destinations](transfer.md#transfer-targets)).

| Parameter | Required | Description |
|---|---|---|
| `camp` | ✓ | Campaign ID (you must own it) |
| `bot` | ✓ | The ID of your active bot that administers the campaign's chat, different from the current one |

```json
{ "status": "ok", "media_lost": false }
```

`media_lost` is `true` if the welcome media couldn't be moved (re-upload it).

| Error | Meaning |
|---|---|
| `func` | `camp` or `bot` not passed, or the same bot given (a no-op move) |
| `access` | The campaign isn't yours |
| `held` | The campaign or its current bot is frozen by moderation |
| `notadmin` | The target bot is not an active administrator of the chat |
| `db` | Database or transaction error |
