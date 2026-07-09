# Media Buying Agent — Meta Ads

A three-skill workflow for running weekly Meta paid social performance
reviews and, when action is warranted, executing the resulting budget and
kill decisions — gated behind an explicit confirmation step.

## Skills

| Skill | Role |
|---|---|
| `mba-meta` | Orchestrator. Resolves the active brand config, runs the reviewer, decides whether any action is needed, and hands off to the committer if so. |
| `mba-meta-performance-reviewer` | Pulls performance data, computes CAC/ROAS at campaign/adset/ad level, and renders a three-section action list (increase / decrease / kill). |
| `mba-meta-commit` | Builds a confirmation table from the reviewer's output, waits for explicit user sign-off, then executes approved changes on Meta. |

Each skill is account-agnostic — every brand-specific value (market, ad
account ID, currency, thresholds) comes from a config file, never from
hardcoded values in the skill instructions.

## Prerequisites

Two MCP connections need to be active before this workflow will run:

- **Meta Ads MCP** — required by the reviewer (read-only entity/status
  checks) and the committer (budget updates, ad pauses). Must expose
  `ads_get_ad_entities` and `ads_update_entity`.
- **Kickbite Marketing MCP** — required by the reviewer to pull performance
  data via `kickbite_marketing_channels_deepdive`. This is Kickbite's
  internal analytics connector — it is not something an account outside
  Kickbite will have access to.

## Setting up a new brand

1. Copy `assets/config.example.json` (found in any of the three skill
   folders) to your own working folder, renamed to `<brand>.config.json`.
2. Fill in `account_name`, `website_id`, `channel`, `ad_account_id`,
   `currency`, and `thresholds`. Omitted thresholds fall back to documented
   defaults — the skills will say so explicitly in their output rather than
   silently assuming a number.
3. Keep this file in whatever folder you're working from for that brand.
   **Do not commit brand config files to this repository** — they identify
   a real ad account and belong with the person running the workflow, not
   in version control.

## Run history (`log.json`)

The reviewer writes one entry per run to a `log.json` file in the same
folder as the config; the committer updates that entry with results after
execution. This file accumulates real performance and account data over
time — keep it local to each brand's working folder. **Never commit
`log.json` to this repository.**

## Safety

The committer always renders a confirmation table (Type | Name | Budget
now | Budget new | Kill) and waits for an explicit yes / no / modify before
calling any Meta write endpoint. No budget change or ad pause happens
without that step, and a decline or partial acceptance is recorded in the
log rather than silently dropped.

## Known limitations

- Tool calls to the Meta Ads MCP are currently referenced by a specific
  connector instance ID rather than resolved dynamically by tool name. This
  means the skills as written are tied to one specific Meta Ads connection
  and won't automatically pick up a different install's connector. Fixing
  this (resolve via `ToolSearch` by tool name, then call whatever full name
  comes back) is a known next step before sharing this more broadly.
- The Kickbite Marketing MCP dependency means this workflow only works for
  brands whose performance data is queryable through Kickbite's internal
  analytics stack. It is not yet a general-purpose plugin usable outside
  Kickbite without replacing that data layer.
