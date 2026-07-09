# Media Buying Agent — Meta Ads

A four-skill workflow for running weekly Meta paid social performance
reviews and, when action is warranted, executing the resulting budget and
kill decisions — gated behind an explicit confirmation step. A brand with no
config file yet is onboarded automatically rather than failing or requiring
someone to hand-write JSON.

## Skills

| Skill | Role |
|---|---|
| `mba-onboarding` | Runs automatically the first time a brand has no config file. Resolves the Meta ad account and Kickbite market/store, resolves targets, collects thresholds, and writes the new config + log file. |
| `mba-meta` | Orchestrator. Resolves the active brand config (handing off to `mba-onboarding` if none exists), runs the reviewer, decides whether any action is needed, and hands off to the committer if so. |
| `mba-meta-performance-reviewer` | Pulls performance data, computes CAC/ROAS at campaign/adset/ad level, and renders a three-section action list (increase / decrease / kill). |
| `mba-meta-commit` | Builds a confirmation table from the reviewer's output, waits for explicit user sign-off, then executes approved changes on Meta. |

Each skill is account-agnostic — every brand-specific value (market, ad
account ID, currency, targets, thresholds) comes from a config file, never
from hardcoded values in the skill instructions.

## Prerequisites

Two MCP connections need to be active before this workflow will run:

- **Meta Ads MCP** — required by the reviewer (read-only entity/status
  checks), the committer (budget updates, ad pauses), and onboarding
  (listing ad accounts). Must expose `ads_get_ad_entities`,
  `ads_update_entity`, and `ads_get_ad_accounts`.
- **Kickbite Marketing MCP** — required by the reviewer to pull performance
  data via `kickbite_marketing_channels_deepdive`, and by onboarding to
  resolve a market/store (`kickbite_list_markets`,
  `kickbite_list_dimension_values`) and read any existing targets
  (`kickbite_get_targets`, read-only — onboarding never calls
  `kickbite_set_targets`). This is Kickbite's internal analytics connector —
  it is not something an account outside Kickbite will have access to.

## Setting up a new brand

**Automatic (recommended):** just run `mba-meta` or
`mba-meta-performance-reviewer` for a brand that has no config yet. Their
Step 0 detects the missing config and hands off to `mba-onboarding`, which
walks through picking the Meta ad account and Kickbite market/store,
resolving targets, and setting thresholds — then writes the config and log
file for you. Onboarding only fires when no config file is found; a missing
`log.json` alone is normal on a brand's first run and is handled by the
reviewer, not by onboarding.

**Manual (fallback):** if you'd rather hand-write the file:

1. Copy `assets/config.example.json` (found in any of the four skill
   folders) to your own working folder, renamed to `<brand>.config.json`.
2. Fill in `account_name`, `website_id`, `channel`, `ad_account_id`,
   `currency`, `targets`, and `thresholds`. Omitted thresholds fall back to
   documented defaults — the skills will say so explicitly in their output
   rather than silently assuming a number.
3. Keep this file in whatever folder you're working from for that brand.
   **Do not commit brand config files to this repository** — they identify
   a real ad account and belong with the person running the workflow, not
   in version control.

## Run history (`log.json`)

The reviewer writes one entry per run to a `log.json` file in the same
folder as the config; the committer updates that entry with results after
execution. `log.json` is shared per working folder, not per brand — if
several brands' config files live in the same folder, their entries all
land in the same `log.json`, disambiguated by each entry's `account` field.
Onboarding creates this file only if it doesn't already exist, so a new
brand added to a folder that already has one never wipes another brand's
history. This file accumulates real performance and account data over
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
