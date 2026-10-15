# Profile & tracker

View account data, rotate the API key, configure the global postback templates,
choose the interface language and notifications, and connect the AlterCPA Lite tracker.

All responses are **HTTP 200** with `Content-Type: application/json`; the result is
determined by the `status` field, not the HTTP code. Success is returned as
`{"status":"ok", …}` with fields **at the top level** (no `data` wrapper); failure as
`{"status":"error","error":"<code>"}`. Timestamps are Unix-second integers, `0` when
unset. The shared error codes are described in [Authentication & conventions](authentication.md).

Every endpoint on this page works with your API key and does not require a separate subscription.

---

<a id="profile-info"></a>

## Profile data

`GET /api/profile/info.json`

The full account card: identity, API key, creation and subscription-end dates, the
granted tariff with its limits and feature flags, current usage, the five global
postback URL templates, and notification settings.

```json
{
  "status": "ok",
  "id": 42,
  "login": "johndoe",
  "name": "John Doe",
  "lang": "en",
  "api_key": "d3adb33fcafe1234567890abcdef0123456789ab",
  "created": 1751328000,
  "paid_until": 1782950400,
  "active": true,
  "tariff": {
    "id": 2,
    "code": "cat",
    "name": "Kitty",
    "campaigns": 15,
    "broadcasts": true,
    "broadcasts_day": 1350,
    "screens": true,
    "crm": false,
    "operators": 0
  },
  "usage": {
    "campaigns": 3
  },
  "postbacks": {
    "wait": "",
    "hold": "",
    "approve": "https://www.altercpa.red/postbacks/abc?click={click}&uid={lead}&name={username}&status=approve",
    "cancel": "https://www.altercpa.red/postbacks/abc?click={click}&uid={lead}&name={username}&status=cancel",
    "trash": "https://www.altercpa.red/postbacks/abc?click={click}&uid={lead}&name={username}&status=trash"
  },
  "notify": {
    "tariff": true,
    "leads": true
  }
}
```

`created` and `paid_until` are Unix time (`0` if there were no payments). `active` is
`true` when the subscription is paid and not yet expired.

In the `tariff` object: `campaigns` is the campaign limit (`-1` means unlimited),
`broadcasts` — whether broadcasts are available, `broadcasts_day` — the daily limit on
manual broadcasts, `screens` — whether multi-screen welcomes are available, `crm` —
whether CRM is available, `operators` — the CRM operator limit.

`usage.campaigns` is the number of non-deleted campaigns. In the `postbacks` object an
empty field means the postback for that event is globally disabled. `notify.tariff` —
whether tariff-expiry warnings are enabled, `notify.leads` — the global toggle for lead
notifications.

| Error | Meaning |
|---|---|
| `db` | Database error while reading the profile |

---

<a id="profile-apikey"></a>

## Regenerate API key

`POST /api/profile/apikey.json`

Generates a new eternal public API key. The old key stops working immediately. No
request body is required.

```json
{ "status": "ok", "api_key": "9f8e7d6c5b4a39281706f5e4d3c2b1a09f8e7d6c" }
```

The `api_key` field contains the bare 40-character token (without the `{user_id}-`
prefix). For authentication you prepend your ID: `42-9f8e7d6c…`.

| Error | Meaning |
|---|---|
| `db` | Database error while updating the key |

---

<a id="profile-postbacks"></a>

## Set global postbacks

`POST /api/profile/postbacks.json`

Sets the five global postback URL templates that campaigns use by default. **Partial
update** — only the supplied fields are written; an omitted field is left unchanged.
Pass an empty string to clear a template. If no field is supplied, this is a safe no-op
that returns `{"status":"ok"}`.

| Parameter | Description |
|---|---|
| `wait` | URL template for the **wait** event (bot link opened) |
| `hold` | URL template for the **hold** event (join request submitted) |
| `approve` | URL template for the **approve** event (joined or request approved) |
| `cancel` | URL template for the **cancel** event (left the chat voluntarily, within the window) |
| `trash` | URL template for the **trash** event (kick or ban, within the window) |

The available `{macros}` for templates are described in [Tracker integration](https://docs.altercpa.top/en/trackers/).

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `db` | Database error while updating the templates |

---

<a id="profile-notify"></a>

## Set notifications

`POST /api/profile/notify.json`

Sets the notification preferences. **Partial update** — omitted fields are left
unchanged.

| Parameter | Description |
|---|---|
| `tariff` | bool: `true` = receive tariff-expiry warnings. The payment notification is always sent. |
| `leads` | bool: `true` = global toggle for lead notifications from all campaigns. |

Boolean values are accepted as `1`/`true`/`yes`/`on` (otherwise off).

```json
{ "status": "ok" }
```

| Error | Meaning |
|---|---|
| `db` | Database error while updating the settings |

---

<a id="profile-lang"></a>

## Set language

`POST /api/profile/lang.json`

Sets the interface language. The change applies immediately.

| Parameter | Required | Description |
|---|---|---|
| `lang` | ✓ | Exactly `en` or `ru`; any other value is a `func` error |

```json
{ "status": "ok", "lang": "ru" }
```

| Error | Meaning |
|---|---|
| `func` | `lang` is not `en` or `ru` |
| `db` | Database error while updating |

---

<a id="profile-tracker"></a>

## Tracker integration status

`GET /api/profile/tracker.json`

Returns the tracker connection status. The saved tracker key is never returned.

```json
{ "status": "ok", "connected": true, "domain": "www.altercpa.red" }
```

`connected` is `true` if the tracker domain is set. `domain` is an empty string when
the tracker is not connected.

| Error | Meaning |
|---|---|
| `db` | Database error while reading the status |

---

<a id="profile-tracker-set"></a>

## Connect / disconnect the tracker

`POST /api/profile/tracker.json`

Connects (or reconnects) or disconnects the AlterCPA Lite tracker. The full guide is in
[AlterCPA Lite integration](https://docs.altercpa.top/en/trackers/altercpa-lite/).

**Connect:** pass `domain` + `key`. The service validates the credentials by calling
the tracker's own endpoint `GET /api/tracker/status.json` on the given domain (this
endpoint belongs to your tracker, not to our API), obtains the postback token, and
auto-fills all five global postback templates.

**Disconnect:** pass `domain=""`. This clears the tracker domain and key; the postback
templates are **kept** and continue to fire until you edit or clear them manually.

| Parameter | Required | Description |
|---|---|---|
| `domain` | ✓ | The tracker domain, e.g. `www.altercpa.red`. Trailing `/` are trimmed. Must not contain `/`, `?` or `#` (otherwise `func`). An empty string means disconnect. |
| `key` | | The tracker API key (format `{user_id}-{user_api}` on the tracker side). Required when connecting (non-empty domain); an empty key with a non-empty domain is a `func` error. |

Connect succeeded:

```json
{
  "status": "ok",
  "connected": true,
  "domain": "www.altercpa.red",
  "postback_sample": "https://www.altercpa.red/postbacks/abc?click={click}&uid={lead}&name={username}&status=approve"
}
```

`postback_sample` is the generated postback template for the **approve** event (shown
on the profile screen as an example of the saved URL).

Disconnect succeeded:

```json
{ "status": "ok", "connected": false }
```

If the validation request to the tracker fails (domain unreachable, bad credentials, or
an empty token), the service returns a **success** envelope with an `error` field rather
than the standard error — the `status` field stays `ok`:

```json
{ "status": "ok", "error": "connect" }
```

| Error | Meaning |
|---|---|
| `func` | The domain contains `/`, `?` or `#`, or no `key` was supplied when connecting |
| `busy` | The validation-request rate limit was exceeded (15 per minute per user) |
| `db` | Database error while saving the tracker settings |

In addition, a failed tracker validation returns `{"status":"ok","error":"connect"}`
(see above) — this is not the standard error envelope.
