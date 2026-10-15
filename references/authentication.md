# Authentication & conventions

This page describes the conventions every other API section relies on: how to
authenticate, how the response format is structured, and which system errors any
endpoint may return.

<a id="auth"></a>

## Authentication

A request can be authenticated in two ways; both resolve to the same user and go
through the same ban check.

<a id="api-key"></a>

### API key (scripts, recommended)

Your permanent API key lives in [Profile → API key](https://docs.altercpa.top/en/guide/profile/).
Format: `{user_id}-{user_api}`, for example `42-d3adb33f…`.

Pass it as the query parameter `id` — this works on every endpoint, including
POST (the query string is always read, even when the request body is JSON):

```bash
curl "https://my.altercpa.top/api/profile/info.json?id=42-d3adb33f…"
```

Or in the `Authorization` header as a Bearer token (in this case only the key
itself is passed, without `{user_id}-`):

```bash
curl -H "Authorization: Bearer d3adb33f…" \
     "https://my.altercpa.top/api/profile/info.json"
```

When the `id` parameter has the form `{user_id}-{user_api}`, it is treated
**only** as an API key, and `user_id` must match the key's owner — otherwise the
`key` error is returned. If `id` does not look like `{user_id}-{user_api}` (for
example, it is a resource identifier), it is ignored for authentication and the
Bearer token from the header is applied instead.

The API key never expires. You can rotate it in your [Profile](https://docs.altercpa.top/en/guide/profile/)
or via [`profile/apikey`](profile.md#profile-apikey); the old key stops
working immediately.

The dashboard signs in through Telegram and uses its own short-lived session
token; for programmatic access, use the API key above.

---

<a id="conventions"></a>

## Conventions

- **GET to read, POST to write.** No exceptions.
- Every response is **HTTP 200** with `Content-Type: application/json`, regardless
  of the outcome. Check the `status` field, not the HTTP code.
- A successful response is almost always **flat**: fields sit at the top level
  next to `"status": "ok"`.

  ```json
  { "status": "ok", "id": 42, "api_key": "42-d3adb33f…" }
  ```

- Some endpoints (typically lists) wrap their payload in a `data` key:

  ```json
  { "status": "ok", "data": [ … ] }
  ```

  Which format a given endpoint uses is indicated in its response example.
- An error is always flat:

  ```json
  { "status": "error", "error": "code" }
  ```

- Timestamps in responses are integers (Unix seconds); `0` means the value is not
  set.
- Endpoints end in `.json` (for example `profile/info.json`).
- The body of POST requests may be JSON (`Content-Type: application/json`) or
  form-encoded (`application/x-www-form-urlencoded`). With a JSON body, its values
  take precedence over query parameters of the same name. The body size is limited
  to 1 MiB (except for file uploads, which have a separate limit).
- The `?id=` authentication parameter is always passed in the **query string**,
  even for POST endpoints.

---

<a id="errors"></a>

## System errors

These errors may be returned by any authenticated endpoint.

| Error | Meaning |
|---|---|
| `key` | Missing or invalid API key |
| `access` | Authentication succeeded, but the resource is not yours or does not exist |
| `ban` | The account is blocked |
| `func` | Unknown endpoint path or wrong method |
| `db` | Database error — retry; if the problem persists, contact support |
| `unpaid` | The subscription has expired and the grace period has ended |
| `limit` | A tariff or quota limit has been exceeded |

Errors specific to particular endpoints (`exists`, `token`, `webhook`,
`unreachable`, …) are described on the corresponding pages.

**Grace period:** when a subscription lapses, the public API keeps working for a
grace window (currently 7 days, though the operator can change this period); after
that, authenticated product endpoints return `unpaid`. Login, profile and billing
endpoints keep working throughout, so that an expired account can still sign in,
check its status and renew the subscription. The full transition rules are in
[Billing & tariffs](https://docs.altercpa.top/en/guide/billing/).
