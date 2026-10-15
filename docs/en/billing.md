# Tariffs & billing

Browse the available tariffs, create and follow a payment through, and read your
payment history. Payments are processed through AffiCat Pay. For more on tariffs,
renewal and the grace period, see [Billing & tariffs](https://docs.altercpa.top/en/guide/billing/).

Every endpoint in this section requires authentication (your API key, see
[Profile & API key](profile.md#profile-apikey)). Billing endpoints work
**without** an active subscription — a lapsed account can still pay for a tariff.

Every response is HTTP 200 with `Content-Type: application/json`. Success or failure
is determined by the `status` field in the body, not by the HTTP code. An error always
comes back as `{"status":"error","error":"<code>"}`. Time fields are integer Unix
seconds (`0` when unset), not date strings.

---

<a id="tariff-list"></a>

## List tariffs

`GET /api/tariff/list.json`

Returns all active tariffs in the order set by the administrator. Each card is
annotated with a charge preview (proration): how many days buying this particular
tariff right now would grant, taking the remainder of the current subscription into
account.

```json
{
  "status": "ok",
  "data": [
    {
      "id": 1,
      "code": "kitten",
      "name": "Kitten",
      "campaigns": 1,
      "price_usd": 0,
      "price_rub": 0,
      "days": 30,
      "broadcasts": false,
      "broadcasts_day": 0,
      "screens": false,
      "crm": false,
      "operators": 0,
      "until": 1748692800,
      "get_days": 30,
      "credit_days": 0
    },
    {
      "id": 2,
      "code": "cat",
      "name": "Kitty",
      "campaigns": 15,
      "price_usd": 24,
      "price_rub": 2400,
      "days": 30,
      "broadcasts": true,
      "broadcasts_day": 1350,
      "screens": true,
      "crm": false,
      "operators": 0,
      "until": 1751284800,
      "get_days": 33,
      "credit_days": 3
    },
    {
      "id": 5,
      "code": "cattery",
      "name": "Cattery",
      "campaigns": 100,
      "price_usd": 135,
      "price_rub": 13500,
      "days": 30,
      "broadcasts": true,
      "broadcasts_day": 13500,
      "screens": true,
      "crm": true,
      "operators": 15,
      "until": 1751284800,
      "get_days": 31,
      "credit_days": 1
    }
  ]
}
```

`data` is an array (empty `[]` when there are no tariffs).

| Field | Meaning |
|---|---|
| `id` | Numeric tariff identifier. This is what you pass to [tariff/buy](billing.md#tariff-buy) |
| `code` | Machine code of the tariff |
| `name` | Tariff name, localised to the user's language. The English name is returned only when the language is `en` and the tariff has its English field filled in; otherwise the Russian name is returned (Russian is the default) |
| `campaigns` | Campaign limit; `-1` means unlimited |
| `price_usd` | Price in USD |
| `price_rub` | Price in RUB |
| `days` | Subscription term in days |
| `broadcasts` | Whether broadcasts are available |
| `broadcasts_day` | Daily broadcast message limit (`0` when broadcasts are unavailable) |
| `screens` | Whether multi-screen welcomes are available |
| `crm` | Whether the operator CRM chat is available |
| `operators` | Number of CRM operator seats (`0` when CRM is unavailable) |
| `until` | Projected subscription expiry if you buy the tariff now (Unix time) |
| `get_days` | Total number of days the purchase would grant, accounting for the remainder of the current subscription |
| `credit_days` | How many of those days are credited from the current subscription's remainder (proration) |

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="tariff-buy"></a>

## Buy a tariff

`POST /api/tariff/buy.json`

Creates a payment for a tariff. A paid tariff creates an invoice in AffiCat Pay and
returns the payment-page URL — redirect the user to it. A free tariff (price `0` in
both currencies) is activated instantly, without contacting the gateway, and no URL
is returned.

| Parameter | Required | Description |
|---|---|---|
| `tariff` | ✓ | Numeric tariff identifier — the `id` field from the [tariff list](billing.md#tariff-list). This is a number, not the machine code |

Paid tariff — the response contains `pay_id` and the payment-page `url`:

```json
{
  "status": "ok",
  "pay_id": 55,
  "url": "https://pay.affi.cat/pay/abc123"
}
```

Free tariff — the invoice is marked paid right away, no URL is needed:

```json
{
  "status": "ok",
  "pay_id": 56,
  "free": true,
  "paid": true
}
```

`url` is an opaque link to the payment page returned by the gateway; the exact host
may differ, so do not hard-code it.

After payment, the subscription is extended automatically once the gateway confirms
the payment. Payment can also be confirmed via [pay/check](billing.md#pay-check)
when the user returns from the payment page — both paths are idempotent.

**Stacking.** Paid tariffs are extended from the current expiry date, not from today:
on an early renewal or a switch, the remainder is recalculated into days of the new
tariff (see the `get_days`/`credit_days` fields). A free tariff, however, is set from
today and does not stack, so it would override a still-running paid period — that
switch is blocked (error `active`, see below).

| Error | Meaning |
|---|---|
| `func` | The `tariff` parameter is missing, equals `0`, or points to a non-existent/hidden tariff |
| `busy` | Invoice-creation limit exceeded (no more than 20 per 60 seconds per user) |
| `active` | An attempt to take a **free** tariff while a running paid plan has more than 5 days left. Paid renewals/switches are not blocked by this rule |
| `pay` | Failed to create the invoice in AffiCat Pay (the dangling invoice is marked failed) |
| `db` | Database error |

---

<a id="pay-resume"></a>

## Resume payment

`POST /api/pay/resume`

Reopens the payment page for your own unfinished (pending) payment and returns its
URL. Lets you complete an abandoned payment without creating a duplicate: the gateway
matches the invoice by store and `uid`, so the same `pay_id` always leads to the same
form.

| Parameter | Required | Description |
|---|---|---|
| `uid` | ✓ | The `pay_id` of an existing pending payment that belongs to you |

```json
{
  "status": "ok",
  "pay_id": 55,
  "url": "https://pay.affi.cat/pay/abc123"
}
```

| Error | Meaning |
|---|---|
| `func` | The `uid` parameter is missing/equals `0`, or the payment is for a free tariff that never had a payment form |
| `busy` | Invoice-creation limit exceeded (shared with [tariff/buy](billing.md#tariff-buy): 20 per 60 seconds) |
| `access` | Payment not found or belongs to another user |
| `status` | The payment is no longer pending (paid, cancelled or refunded) — there is nothing to resume |
| `pay` | Failed to reopen the invoice in AffiCat Pay |
| `db` | Database error |

---

<a id="pay-cancel"></a>

## Cancel payment

`POST /api/pay/cancel`

Marks your own pending payment as failed (status `2`) — the "Cancel" action for an
abandoned invoice on the billing screen. A failed invoice can no longer be turned into
a paid one, so a late postback on a cancelled invoice is safely ignored.

| Parameter | Required | Description |
|---|---|---|
| `uid` | ✓ | The `pay_id` of your own pending payment |

```json
{
  "status": "ok",
  "pay_id": 55
}
```

| Error | Meaning |
|---|---|
| `func` | The `uid` parameter is missing or equals `0` |
| `access` | Payment not found, belongs to another user, or is no longer pending |
| `db` | Database error |

---

<a id="pay-check"></a>

## Check payment status

`GET /api/pay/check`

Pull-confirmation of your own payment when returning from the payment page. If the
payment is already paid, returns `paid: true`; otherwise it polls AffiCat and, if the
invoice is paid, accepts it (extends the subscription). Called automatically on return
and idempotent with the push postback.

| Parameter | Required | Description |
|---|---|---|
| `uid` | ✓ | The `pay_id` of your own payment (the gateway appends it to the return URL) |

```json
{
  "status": "ok",
  "paid": true
}
```

`paid` is `false` if the gateway still reports the invoice as unpaid.

| Error | Meaning |
|---|---|
| `func` | The `uid` parameter is missing or equals `0` |
| `access` | Payment not found or belongs to another user |
| `pay` | Failed to query AffiCat for the invoice status |
| `db` | Database error |

---

<a id="pay-list"></a>

## Payment history

`GET /api/pay/list.json`

Returns your payment history, newest first, with pagination. Internal settlement data
(settled cash, currency, metadata) is not exposed.

| Parameter | Required | Description |
|---|---|---|
| `limit` | | Page size; defaults to `50`, maximum `200` |
| `offset` | | Offset from the start; values below `0` are clamped to `0` |

```json
{
  "status": "ok",
  "data": [
    {
      "id": 55,
      "tariff": 2,
      "usd": 24,
      "rub": 2400,
      "status": 1,
      "created": 1746111600,
      "paid": 1746111720
    }
  ]
}
```

`data` is an array (empty `[]` when there are no payments). The `tariff` field is the
numeric tariff identifier (not a code or a name). The `created` and `paid` fields are
Unix time in seconds; `paid` is `0` for pending and unpaid invoices.

| `status` | Meaning |
|---|---|
| `0` | Pending — the invoice is created, payment not confirmed |
| `1` | Paid |
| `2` | Failed or cancelled — rejected by the gateway, cancelled by the user, or abandoned |
| `3` | Refunded |

| Error | Meaning |
|---|---|
| `db` | Database error |
