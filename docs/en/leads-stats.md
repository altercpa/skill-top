# Leads, journal & stats

View and export leads, read the raw chat activity journal, and fetch aggregated
statistics.

Every endpoint on this page requires an active, paid subscription. If the
subscription has lapsed and the grace period is over, they return the `unpaid`
error. Beyond that, any of them can return the system errors `key`, `ban`, and
`db`; these are described in the
[Authentication & conventions](authentication.md#errors) section.
Endpoint-specific errors are listed in the tables below.

---

<a id="lead-statuses"></a>

## Lead statuses

All lead-status filter and response fields use **numeric codes**.

| Code | Name | Meaning |
|---|---|---|
| `0` | wait | The subscriber opened the bot link but has not joined the chat yet |
| `1` | hold | Sent a join request, awaiting an admin's approval |
| `2` | approve | Joined the chat (or the request was approved) |
| `3` | cancel | Left the chat voluntarily |
| `4` | trash | Removed or banned from the chat |

The full transition rules and postback-firing conditions are in the
[Lead lifecycle](https://docs.altercpa.top/en/guide/leads/) section.

---

<a id="leads-list"></a>

## Leads list

`GET /api/leads/list.json`

Returns the current account's leads matching the filters, newest first.

| Parameter | Required | Description |
|---|---|---|
| `campaign` | | Filter by campaign ID (`0` or absent — all campaigns) |
| `status` | | Filter by lead status code (`0`–`4`). Always applied when the parameter is supplied, including `status=0` (filter by `wait`) |
| `from` | | Start of the lead-created date range, Unix time in seconds (applied when the value is > 0) |
| `to` | | End of the lead-created date range, Unix time in seconds (applied when the value is > 0) |
| `q` | | Subscriber search: an all-digits value is an exact match on subscriber ID; otherwise a substring search on the username (a leading `@` is dropped) |
| `limit` | | Page size (default 50, maximum 200) |
| `offset` | | Pagination offset (a negative value is clamped to 0) |

```json
{
  "status": "ok",
  "data": [
    {
      "id": 1001,
      "campaign": 12,
      "bot": 7,
      "click": "abc456xyz",
      "subscriber": 9876543210,
      "username": "alice",
      "status": 2,
      "start_time": 1747735200,
      "join_time": 1747735260,
      "leave_time": 0,
      "created": 1747735200
    }
  ]
}
```

`data` is an array of leads (an empty `[]` when nothing is found); there is no
`total` field in the response. `username` is an empty string when the subscriber
has no Telegram username. `click` is an empty string when the subscriber arrived
without a click payload (for example, opening the bot link directly rather than
through a deep link). `start_time`, `join_time`, `leave_time`, and `created` are
Unix time in seconds; a value of `0` means the event has not happened yet (for
example, `join_time` is `0` while the lead is in the `wait` or `hold` status).

---

<a id="leads-summary"></a>

## Per-status counts

`GET /api/leads/summary.json`

Returns the lead counts broken down by each status for the current filter. The
`status` filter itself is deliberately ignored here — the full breakdown across
every status is returned, so the interface can show the big picture and let the
user switch between statuses.

| Parameter | Description |
|---|---|
| `campaign` | Filter by campaign ID |
| `from` | Start of the lead-created date range (Unix time in seconds) |
| `to` | End of the lead-created date range (Unix time in seconds) |
| `q` | Subscriber search (a number — exact subscriber ID; otherwise a username substring) |

```json
{
  "status": "ok",
  "total": 540,
  "counts": {
    "0": 50,
    "1": 12,
    "2": 420,
    "3": 35,
    "4": 23
  }
}
```

The `total` and `counts` fields sit at the top level of the response (not inside
a nested `data` object). `counts` is a map where the key is the lead status code
as a string and the value is the number of leads in that status. `total` is the
sum across all statuses.

---

<a id="leads-export"></a>

## Lead export to CSV

`GET /api/leads/export.csv`

Returns the current account's filtered leads as a downloadable CSV file. It
accepts the same filters as the [leads list](leads-stats.md#leads-list)
but without pagination: the entire matching set is exported (no more than 100000
rows).

| Parameter | Description |
|---|---|
| `campaign` | Filter by campaign ID |
| `status` | Filter by lead status code (applied when the parameter is supplied) |
| `from` | Start of the lead-created date range (Unix time in seconds) |
| `to` | End of the lead-created date range (Unix time in seconds) |
| `q` | Subscriber search (a number — exact subscriber ID; otherwise a username substring) |

Unlike the other endpoints, the response is not JSON. It is `text/csv;
charset=utf-8` with a UTF-8 BOM (so Cyrillic in campaign names opens correctly
in Excel) and a `Content-Disposition: attachment; filename="leads-YYYYMMDD.csv"`
header.

The header row and column order:

```
created,status,subscriber,username,campaign,bot,click,started,joined,left
```

- `status` — the word label (`wait`, `hold`, `approve`, `cancel`, `trash`; for
  an unknown code — the number itself).
- `created`, `started`, `joined`, `left` — time in UTC in the
  `YYYY-MM-DD HH:MM:SS` format; an empty string when the event has not happened
  yet.
- `campaign` — the campaign name (if it is empty — its code, otherwise `#<id>`).
- `bot` — the bot's username (if it is empty — `#<id>`).

If the request fails before the file starts streaming, an ordinary JSON error
response is returned (for example, `db`). Once the CSV header has been sent the
response is already committed, so a failure while reading a row simply ends the
stream — the client receives a truncated file.

---

<a id="journal-list"></a>

## Chat activity journal

`GET /api/journal/list.json`

Returns the activity journal for a single chat — every join and leave event,
including **organic** activity (users who joined without launching the bot),
newest first. This is the raw source of channel dynamics; organic events do not
create leads and do not fire postbacks. The endpoint is keyed to a specific
bot + chat pair, so both parameters are required.

| Parameter | Required | Description |
|---|---|---|
| `bot` | ✓ | Bot ID. Must belong to the current account, otherwise the `access` error |
| `chat` | ✓ | Telegram chat ID |
| `action` | | Filter by action type (see the table below). Applied when the parameter is supplied |
| `paid` | | `paid=1` — only entries tied to a lead (`lead` ≠ 0); any other value — only organic entries (`lead` = 0). Applied when the parameter is supplied |
| `from` | | Start of the date range, Unix time in seconds |
| `to` | | End of the date range, Unix time in seconds |
| `limit` | | Page size (default 50, maximum 200) |
| `offset` | | Pagination offset |

```json
{
  "status": "ok",
  "data": [
    {
      "id": 5001,
      "chat": -1001234567890,
      "subscriber": 9876543210,
      "username": "alice",
      "action": 0,
      "lead": 1001,
      "time": 1747735260
    }
  ]
}
```

`data` is an array of entries (an empty `[]` when nothing is found); there is no
`total` field in the response. `action` is a **numeric code** for the action
(see the table below), not a string. `lead` is the ID of the linked lead, or `0`
for organic activity. `time` is Unix time in seconds.

| `action` | Meaning |
|---|---|
| `0` | join — the subscriber joined the chat |
| `1` | leave — the subscriber left the chat voluntarily |
| `2` | kicked — the subscriber was removed or banned |
| `3` | request — the subscriber sent a join request |

| Error | Meaning |
|---|---|
| `func` | `bot` or `chat` was not supplied |
| `access` | The specified bot does not belong to the current account |

---

<a id="journal-export"></a>

## Journal export to CSV

`GET /api/journal/list.csv`

Returns a single chat's activity journal as a downloadable CSV file. It accepts the
same scope and filters as the [chat activity journal](leads-stats.md#journal-list)
(`bot` and `chat` required, optional `action`/`paid`/`from`/`to`) but without
pagination: the entire matching set is exported (no more than 100000 rows).

Unlike the JSON endpoints, the response is `text/csv; charset=utf-8` with a UTF-8 BOM
and a `Content-Disposition: attachment; filename="activity-YYYYMMDD.csv"` header.

The header row and column order:

```
time,subscriber,username,action,type
```

- `time` — the event time in UTC in the `YYYY-MM-DD HH:MM:SS` format.
- `action` — the word label (`join`, `leave`, `ban`, `request`).
- `type` — `paid` if the entry is tied to a lead, otherwise `organic`.

Error behaviour matches the [lead export](leads-stats.md#leads-export): a
failure before streaming starts returns a JSON error (`func`/`access`/`db`), while
after the CSV header is sent the stream simply ends.

| Error | Meaning |
|---|---|
| `func` | `bot` or `chat` was not supplied |
| `access` | The specified bot does not belong to the current account |

---

<a id="stats-summary"></a>

## Summary statistics

`GET /api/stats/summary.json`

Aggregate counters over a period (the last 30 days by default) for the whole
account or a single campaign. Computed in real time from current data: leads are
counted by their **current** status (a snapshot at request time), so the five
statuses sum to the total number of launches. The response is cached for a few
minutes, so the figures refresh with a small delay.

| Parameter | Description |
|---|---|
| `campaign` | Filter by a single campaign (omit for account-wide statistics). If a campaign is specified but does not belong to the account — the `access` error |
| `from` | Start of the period, Unix time in seconds (default — midnight 29 days ago; the window covers the last 30 days, including today) |
| `to` | End of the period, Unix time in seconds (default — the current moment) |

```json
{
  "status": "ok",
  "totals": {
    "wait": 50,
    "hold": 12,
    "approve": 420,
    "cancel": 35,
    "trash": 23,
    "org_join": 150,
    "org_leave": 42
  },
  "starts": 540,
  "confirms": 420,
  "unsubs": 35,
  "lefts": 23,
  "cr": 0.7777777777777778
}
```

All fields sit at the top level of the response (not inside a nested `data`
object). `totals` is the seven base counters: the five lead statuses (`wait`,
`hold`, `approve`, `cancel`, `trash`) plus organic `org_join` and `org_leave`.
The derived fields:

- `starts` — the sum of the five lead statuses (`wait` + `hold` + `approve` +
  `cancel` + `trash`).
- `confirms` — equals `totals.approve`.
- `unsubs` — equals `totals.cancel`.
- `lefts` — equals `totals.trash`.
- `cr` — the conversion rate as a **fraction between 0 and 1** (`approve /
  starts`), without multiplying by 100 and without rounding (for example,
  `0.84`). If `starts` is 0, `cr` is 0.

| Error | Meaning |
|---|---|
| `access` | A campaign is specified but does not belong to the current account |

---

<a id="stats-series"></a>

## Daily time series

`GET /api/stats/series.json`

Per-day counters for building charts. It accepts the same filters as the
[summary statistics](leads-stats.md#stats-summary).

| Parameter | Description |
|---|---|
| `campaign` | Filter by a single campaign. If a campaign is specified but does not belong to the account — the `access` error |
| `from` | Start of the period, Unix time in seconds (default — the last 30 days) |
| `to` | End of the period, Unix time in seconds (default — the current moment) |

```json
{
  "status": "ok",
  "series": [
    { "date": "2026-05-20", "wait": 5, "hold": 1, "approve": 19, "cancel": 2, "trash": 0, "org_join": 8, "org_leave": 3 },
    { "date": "2026-05-21", "wait": 4, "hold": 0, "approve": 15, "cancel": 1, "trash": 1, "org_join": 5, "org_leave": 2 }
  ],
  "group": "day"
}
```

`series` is an array of points (one per day; an empty `[]` when there is no
data); each point carries the date and the same seven counters as `totals` in
the summary. `group` is always `"day"`. Days with no activity are omitted. Like
the summary, the series is computed in real time from current data (leads by their
current status) and cached for a few minutes.

| Error | Meaning |
|---|---|
| `access` | A campaign is specified but does not belong to the current account |

---

<a id="stats-campaigns"></a>

## Top campaigns

`GET /api/stats/campaigns.json`

A ranking of campaigns by performance over a period — the "Top campaigns" panel
in the dashboard. It counts leads for each non-deleted campaign of the account
and sorts by the number of confirmed subscribers (`approve`), then by `starts`,
then by ID. Like the summary, it is computed in real time from current data (leads
by their current status) and cached for a few minutes; organic activity does not
take part in the ranking.

| Parameter | Description |
|---|---|
| `from` | Start of the period, Unix time in seconds (default — the last 30 days) |
| `to` | End of the period, Unix time in seconds (default — the current moment) |
| `limit` | Ranking size (default 5; a value ≤ 0 or > 50 is coerced to 5) |

```json
{
  "status": "ok",
  "data": [
    {
      "id": 12,
      "name": "Spring promo",
      "code": "abc123",
      "chat_title": "My Channel",
      "wait": 50,
      "hold": 12,
      "approve": 420,
      "cancel": 35,
      "trash": 23,
      "starts": 540,
      "cr": 0.7777777777777778
    }
  ]
}
```

`data` is an array of campaigns, sorted by `approve` descending, then `starts`,
then ID ascending, and truncated to `limit`. For each campaign: `wait`, `hold`,
`approve`, `cancel`, `trash` are the lead-status counters; `starts` is their sum;
`cr` is the conversion rate as a fraction between 0 and 1 (`approve / starts`),
as in the summary. There is no separate campaign filter here — the ranking is
built across all non-deleted campaigns of the account.

---

<a id="stats-report"></a>

## Grouped report

`GET /api/stats/report.json`

The grouped statistics table — the "Statistics" screen in the dashboard. It counts
launches and leads by status (from current data, leads by their current status) plus
chat activity (joins, leaves, bans), grouped by the chosen dimension.

| Parameter | Description |
|---|---|
| `group` | Grouping dimension: `date` (default), `campaign`, or `bot` |
| `traffic` | Which traffic to count in the activity columns: `all` (default), `organic` (organic only), or `campaign` (campaign-driven only). Affects **only** `joins`/`leaves`/`bans`, not the status counters |
| `from` | Start of the period, Unix time in seconds (default — the last 30 days) |
| `to` | End of the period, Unix time in seconds (default — the current moment) |
| `bots` | Comma-separated list of bot IDs — a multi-bot filter |
| `campaigns` | Comma-separated list of campaign IDs — a multi-campaign filter |

```json
{
  "status": "ok",
  "group": "date",
  "traffic": "all",
  "rows": [
    {
      "key": "2026-06-20",
      "label": "2026-06-20",
      "total": 540,
      "valid": 517,
      "wait": 50,
      "hold": 12,
      "approve": 420,
      "cancel": 35,
      "trash": 23,
      "cr": 0.7777777777777778,
      "validity": 0.9574074074074074,
      "joins": 95,
      "leaves": 18,
      "bans": 4
    }
  ],
  "totals": {
    "key": "",
    "label": "",
    "total": 540,
    "valid": 517,
    "wait": 50,
    "hold": 12,
    "approve": 420,
    "cancel": 35,
    "trash": 23,
    "cr": 0.7777777777777778,
    "validity": 0.9574074074074074,
    "joins": 95,
    "leaves": 18,
    "bans": 4
  }
}
```

| Field | Meaning |
|---|---|
| `key` | The group key: a date (`YYYY-MM-DD`) or a stringified campaign/bot ID |
| `label` | The display label for the group (date / campaign name / `@bot`) |
| `code` | The campaign code (present only when `group=campaign`) |
| `total` | All leads (= sum of the five statuses, i.e. all `/start` launches) |
| `valid` | All leads except `trash` (`total − trash`) |
| `wait`…`trash` | Lead-status counters |
| `cr` | `approve / total`, a fraction between 0 and 1 |
| `validity` | `valid / total`, a fraction between 0 and 1 |
| `joins` / `leaves` / `bans` | Chat activity: joins / voluntary leaves / bans (kept separate, unlike the rollups) |

`rows` are sorted by date (newest first), and for the campaign/bot grouping by `total`
descending, then `approve`, then ID. `totals` carries the same metrics summed across
all rows (its `key`/`label` are empty).

| Error | Meaning |
|---|---|
| `db` | Database error |

---

<a id="stats-report-csv"></a>

## Grouped report as CSV

`GET /api/stats/report.csv`

The same data as the [grouped report](leads-stats.md#stats-report), with the
same parameters (`group`/`traffic`/`from`/`to`/`bots`/`campaigns`), as a downloadable
CSV file. The response is `text/csv; charset=utf-8` with a UTF-8 BOM and a
`Content-Disposition: attachment; filename="stats-YYYYMMDD.csv"` header.

The first column is the grouping dimension (`date`, `campaign`, or `bot`; for the
campaign grouping a `code` column is added), then
`total,valid,wait,hold,approve,cancel,trash,cr,validity,joins,leaves,bans`. The last
row is a `total` row with the sums across all groups.

| Error | Meaning |
|---|---|
| `db` | Database error |
