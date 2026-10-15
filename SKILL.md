---
name: altercpa-top-api
description: "AlterCPA TOP is a Telegram join-tracking service (bots, campaigns, postbacks to trackers). Use this skill for the TOP user API (my.altercpa.top/api) - bots, campaigns, welcome screens and media, broadcasts, CRM teams and operators, leads export, stats, transfer, billing, API key, global postbacks."
---

# AlterCPA TOP API

TOP links Telegram bots to your channels and groups. A *campaign* is a bot plus a chat: it greets whoever opens its
deep link, tracks each lead (`0` wait, `1` hold, `2` approve, `3` cancel, `4` trash) and fires postbacks to a tracker.

## Basics

- Base URL `https://my.altercpa.top/api`. GET reads, POST writes, except `GET pay/check` (confirms a payment) and the
  first `GET transfer/token` (creates the token). Append `.json` to calls below, except `.csv` ones and
  `pay/{resume,cancel,check}`. See `references/authentication.md`.
- Auth: `?id={user_id}-{user_api}` (query string, also for POST) or `Authorization: Bearer {user_api}`.
- Always HTTP 200 + JSON; check `status`. Success is flat `{"status":"ok",...}` or has `"data":[...]`; error is
  `{"status":"error","error":"<code>"}`. Times are Unix seconds, `0` = unset.
- Any call may fail with `key access ban func db unpaid limit`. `unpaid` = no active paid subscription once the grace period (7 days by default) is over (not needed by
  profile, billing, CRM, transfer token/resolve).
- Some writes need tariff features (errors `broadcasts`, `screens` incl. CAS and cleanup, `crm`); switching off is always allowed.

## Endpoints

Per call: `references/<page>.md`. `{a,b}` expands to calls; purposes follow the same order, split by `/`.

| Call | Purpose |
|---|---|
| **profile.md** |
| GET profile/info | Account card |
| POST profile/postbacks | Global postback templates, one per status |
| POST profile/{apikey,notify,lang} | Rotate API key / notifications `tariff`,`leads` / language `en` or `ru` |
| GET,POST profile/tracker | Status / connect AlterCPA Lite (`domain`,`key`) or disconnect (`domain=""`) |
| **bots.md** |
| GET bots/{list,chats,campaigns} | Bots / free admin chats / bot's campaigns (`id`) |
| POST bots/{add,default,del,photo} | Connect by BotFather `token` / pin default campaign (`id`,`campaign`) / disconnect / avatar (multipart `file`) |
| GET,POST bots/profile | Read / save `name`,`description`,`short_description` |
| **campaigns.md** |
| GET camp/{list,get,link,flowstats,clean} | All / one / deep link / tracker stats / cleanup progress |
| POST camp/{add,edit,del,pause,mode,cas,flow,flowsync,clean} | Create (`bot`,`chat`) / edit / soft-delete / pause (`pause=1`) or resume / welcome mode `0` classic `1` screens / CAS / create flow / restore it / start cleanup |
| GET,POST camp/broadcast | Triggered blocks / save one (also in broadcasts.md) |
| POST camp/btest | Preview a block to yourself |
| GET screen/list | Screens with buttons (`camp`) |
| POST screen/{add,edit,copy,del,reorder,test} | Add / edit / duplicate / delete / order / preview |
| POST button/{add,edit,del,reorder} | Add / replace / delete / order |
| POST media/upload | Photo/video, returns `media`,`media_type` |
| **broadcasts.md** |
| GET broadcast/{list,quota,get} | Manual broadcasts / daily quota / one |
| POST broadcast/{add,edit,start} | Draft / body, filters, scope / queue audience |
| POST broadcast/{pause,resume,reset,cancel,del,test} | Pause / resume / back to draft / stop / delete / preview |
| **crm.md** |
| GET crm/{bot,team,operator,template,event}/list | Bots / teams / operators+seats / sets+templates / events (`bot`) |
| POST crm/{bot,team,operator,template,set,event} | CRM per bot (`bot`,`crm`) / team / operator (one-time `token`) / template / set / event; `id=0` creates |
| POST crm/{team,operator,template,set,event}/del | Delete (a team must have no operators) |
| POST crm/{team/bots,team/sets,operator/status,operator/reset} | Replace bot allowlist / shared template sets / enable `0` or disable `1` operator / new one-time token |
| GET crm/{stats,dialog/list} | Stats / open dialogs (`before`,`filter`,`bot`) |
| POST crm/dialog/assign | Reassign / return to pool (`dialog`,`operator`) |
| **leads-stats.md** |
| GET leads/{list,summary,export.csv} | Leads / counts per status / CSV |
| GET journal/{list,list.csv} | Join/leave journal (`bot`,`chat`) / CSV |
| GET stats/{summary,series,campaigns,report,report.csv} | Totals / per day / top campaigns / report by `group` date, campaign or bot / CSV |
| **transfer.md** |
| GET transfer/{token,resolve,targets} | Your transfer token / check recipient `token` / your other bots for a campaign |
| POST transfer/{token,bot,camp,move} | Regenerate token / hand over bot / campaign / move campaign to your other bot |
| **billing.md** |
| GET tariff/list | Plans, prices, proration |
| POST tariff/buy | Pay for a plan (`tariff` id), returns `url` |
| POST pay/{resume,cancel} | Reopen / cancel a pending payment (`uid`) |
| GET pay/{check,list} | Confirm a payment (`uid`) / history |

## Gotchas

- `id` is also the resource id: unless `{user_id}-{key}`-shaped it is ignored for auth and the Bearer header is used.
- Soft errors: `status: ok` plus `error` (`connect`, `create`, `sync`, `noflow`, `stats`) on `profile/tracker` POST and
  `camp/flow*`.
- `camp/add`: `bot`/`chat` come from `bots/chats`; one chat per live campaign; fixed after creation.
- Media: `media/upload` (multipart `file`) first, then `media`+`media_type` (`1` photo, `2` video); `""` removes.
- Transfer: `leads` defaults to `0` = ALL leads deleted for good; pass `leads=1`. Run `transfer/resolve` first.
- Full replace: `button/edit`, `crm/team/bots`, `crm/team/sets`. Flag must be present: `camp/mode`, `camp/cas`, `crm/bot`.
- Broadcasts: `start` tops up unreached leads only; `reset` re-sends to all; cancel a running one before delete.
- Billing: `tariff` is the numeric id; a free plan is refused (`active`) while a paid one has over 5 days; `uid` = `pay_id`.
- `cr` is 0-1 in `stats/*`, a percent in `camp/flowstats`. CSV calls return `text/csv`, not JSON.
- `busy` = rate limit: uploads 20/h, invoices 20/60 s, flow resync 30/min, tracker connect 15/min.
