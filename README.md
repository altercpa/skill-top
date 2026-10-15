# AlterCPA TOP API — a skill for Claude

**Give your AI assistant the complete AlterCPA TOP user API: bots, campaigns, broadcasts, CRM, leads and billing — every endpoint and parameter.**

[![License: MIT](https://img.shields.io/github/license/altercpa/skill-top)](LICENSE)
[![Latest release](https://img.shields.io/github/v/release/altercpa/skill-top)](https://github.com/altercpa/skill-top/releases/latest)

**English** · [Русский](README.ru.md)

![Zlata, the rose-gold cat mascot of AlterCPA TOP — cover illustration for the AlterCPA TOP API skill](.github/assets/cover.jpg)

## What is it

[AlterCPA TOP](https://www.altercpa.top) (Telegram Optimisation Platform) tracks joins to your Telegram
channels and groups through a bot and reports each one as a CPA postback to the advertiser's tracker —
AlterCPA Lite, AlterCPA Pro, Keitaro, Binom or a custom one. This is a [Claude](https://claude.com) **skill** that
teaches Claude the TOP user API. Ask for a script, an integration or a report in plain words — Claude
writes calls with the right endpoints, parameters and error handling instead of guessing.

> This skill is for the **TOP service API** (`my.altercpa.top/api`), not for the API of your tracker.
> The AlterCPA Lite tracker has its own skill: [skill-lite](https://github.com/altercpa/skill-lite).

## What it covers

| Area | What you can do |
|---|---|
| **Profile and tracker** | account data, API key rotation, global postback templates, notifications and language, connecting the AlterCPA Lite tracker |
| **Bots** | connect and disconnect Telegram bots, bot profile and avatar, chats where the bot is an administrator, the default campaign |
| **Campaigns, screens, media** | campaign CRUD, deep links, pause and resume, tracker flows, welcome screens with buttons, media upload, CAS protection, group cleanup |
| **Broadcasts** | triggered follow-ups per lead status; manual bulk broadcasts: create, start, pause, resume, cancel |
| **CRM: teams and operators** | teams, operators, canned replies, custom events, CRM statistics, dialog oversight |
| **Leads, journal, stats** | leads and per-status counts, chat activity journal, summary and grouped statistics, CSV exports |
| **Transfer** | hand a bot with its campaigns, or a single campaign, over to another account |
| **Tariffs and billing** | tariffs, payments (start, resume, cancel, check), payment history |

## Install

### Claude Code

Download [`altercpa-top-api-skill.zip`](https://github.com/altercpa/skill-top/releases/latest/download/altercpa-top-api-skill.zip)
and unpack it into a skills folder:

```sh
unzip altercpa-top-api-skill.zip -d ~/.claude/skills/      # every project
unzip altercpa-top-api-skill.zip -d .claude/skills/        # this project only
```

On Windows unpack it into `%USERPROFILE%\.claude\skills\`.

Prefer git? Clone straight into the skills folder, and `git pull` is the whole update:

```sh
git clone https://github.com/altercpa/skill-top.git ~/.claude/skills/altercpa-top-api
```

### Claude.ai and Claude Desktop

In Claude's settings add a custom skill and upload the same zip.

## Try it

Nothing to call by hand — Claude loads the skill on its own whenever a request touches the
AlterCPA TOP API. For example:

- *Create a campaign for my bot and give me its deep link.*
- *Write a Python script that exports yesterday's leads from AlterCPA TOP to CSV.*
- *List my connected bots.*
- *Set up a broadcast for my campaign's subscribers and start it.*

Most calls follow one pattern; the key is `{user_id}-{user_api}` from your profile in the TOP dashboard:

```
https://my.altercpa.top/api/{area}/{func}.json?id={key}
```

Calls that take their own `id` parameter (such as `camp/get`) send the key as `Authorization: Bearer {user_api}`
instead. A few paths differ — CRM calls have three segments, `pay/*` has no `.json` — and `SKILL.md` lists them all.

## Update

Releases are tagged `vYYYY.MM.DD`. Watch the repository (*Watch → Custom → Releases*) to be
notified, then download the new zip — or run `git pull` if you cloned.

## What is inside

| Path | What |
|---|---|
| [`SKILL.md`](SKILL.md) | the skill's entry point |
| [`references/`](references) | the API reference, one file per API section |

The reference is built from the public API documentation at <https://docs.altercpa.top/en/api/>.

## License

[MIT](LICENSE) — use it, copy it, change it, ship it.

The mascot artwork in [`.github/assets/`](.github/assets) is © AlterCPA and is not covered by the MIT license.
