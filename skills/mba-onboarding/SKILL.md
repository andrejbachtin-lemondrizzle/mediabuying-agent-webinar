---
name: mba-onboarding
description: >
  One-time setup flow for a new brand on the media buying agent. Resolves a
  Meta ad account and a Kickbite market/store via their respective MCPs,
  resolves CAC/ROAS targets (reads Kickbite's existing targets first and only
  asks the user when none are set — this skill never writes targets back to
  Kickbite), collects the six threshold/budget-step values used by the
  reviewer and committer, then writes a new `<account_name>.config.json` and
  an empty `log.json` (if one doesn't already exist) in the current working
  folder.

  Trigger this automatically whenever `mba-meta` or
  `mba-meta-performance-reviewer`'s own Step 0 finds zero matching
  `*.config.json` files — neither of those skills should fall back to an
  ad-hoc question themselves; they hand off here instead, wait for this
  skill to finish, then re-resolve the config and continue their own flow.
  Also trigger directly when the user says things like "set up a new
  brand", "onboard [client]", "connect a new Meta account", "add a new
  client to the media buying agent", or "set up the config for [brand]".
---

# Media Buying Agent — Onboarding

Turns a brand with no config file yet into one that `mba-meta`,
`mba-meta-performance-reviewer`, and `mba-meta-commit` can run against,
without anyone hand-writing JSON. Every value that ends up in the config
comes from a live source (Meta's ad account list, Kickbite's market list,
Kickbite's existing targets) or an explicit user answer — never a guess.

This skill only ever reads from Meta and Kickbite and writes local JSON
files. It does not call `ads_update_entity` or `kickbite_set_targets`.

---

## Step 0 — Confirm onboarding is actually needed

If this skill was invoked directly by the user (not handed off from
`mba-meta` / `mba-meta-performance-reviewer`), check the current working
directory for an existing `*.config.json` yourself first. If one is already
there, say so and ask whether the user wants to onboard an *additional*
brand into this same folder (proceed if yes) or stop — don't silently
overwrite or duplicate an existing brand's config.

If this skill was handed off to because another skill's Step 0 found zero
config matches, proceed directly — that check has already happened.

---

## Step 1 — Select the Meta ad account

Load `mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_get_ad_accounts`
(deferred — load via `ToolSearch` with
`select:mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_get_ad_accounts`),
then call it to list the ad accounts this connection can see.

Present the returned accounts (name, ID, currency) as a numbered list and
ask the user which one this brand is. **Confirm explicitly even if there is
only one account** — unlike config resolution elsewhere in this plugin,
this choice gets written once and silently reused for every future budget
change, so a wrong pick here is a real-money mistake later, not a cosmetic
one.

Capture from the confirmed account:
- `ad_account_id`
- `currency` — read directly from the Meta account object. Do not ask the
  user for this; it must match what Meta's API actually expects for that
  account.

If the call errors or returns nothing, stop and report — don't fall back to
an ID the user mentions in passing without verifying it against this list.

---

## Step 2 — Select the Kickbite market/store and channel

Call `kickbite_list_markets` (load via `ToolSearch` if deferred) and present
the returned markets as a numbered list. Ask the user which market/store
corresponds to the Meta ad account just confirmed.

Capture `website_id` from the selected market.

Then confirm the `channel` value: default to `"Meta Paid"`, but verify it's
a real value for this market by calling `kickbite_list_dimension_values` for
the `channel` dimension. If `"Meta Paid"` isn't in the returned list, show
the actual values and ask the user to pick the right one rather than
guessing a variant spelling.

---

## Step 3 — Confirm the account_name

Suggest an `account_name` slug derived from the Kickbite market name or Meta
account name (lowercase, hyphenated — matching the style in
`assets/config.example.json`, e.g. `example-brand`). Ask the user to confirm
it or give their own.

This value is the key used to match log entries across every future run
(see `mba-meta-performance-reviewer` Step 2/8) and to name the config file
itself (Step 6) — it needs to stay stable for this brand going forward, so
don't let it default silently without a confirmation.

---

## Step 4 — Resolve targets (read-only — never write to Kickbite)

Call `kickbite_get_targets` (load via `ToolSearch` if deferred) scoped to
the `website_id` resolved in Step 2.

- **If it returns an existing target** (e.g. a target CAC or target ROAS
  already set for this market): show it to the user, confirm it's the right
  one to use, and store it in the config with `"source": "kickbite"`.
- **If it returns nothing / no target is set for this market:** ask the
  user directly what their target CAC and/or target ROAS is for this
  account, and store what they give you with `"source": "local"`.

**Never call `kickbite_set_targets`** in this skill, in either branch —
targets are read from Kickbite when they exist, and stored locally in the
brand config when they don't. Pushing a target into Kickbite's own platform
is a separate, higher-blast-radius action outside this skill's scope.

If the user has no target CAC/ROAS in mind, store `null` for both fields
rather than inventing a number — `mba-meta-performance-reviewer` should
treat a null target as "no target configured" in its narrative, not as
zero.

---

## Step 5 — Collect thresholds and budget-step rules

Ask the user for each of the six values below. Offer the documented default
for each (same defaults `mba-meta-performance-reviewer` and
`mba-meta-commit` already fall back to) and let the user accept all six at
once (e.g. "use the defaults") or override individually:

| Field | What it controls | Default |
|---|---|---|
| `thresholds.cac_change_pct_threshold` | % CAC move that triggers INCREASE/DECREASE instead of HOLD | `5` |
| `thresholds.kill_cac_multiplier` | An ad's CAC above this × account L7 CAC is a kill candidate | `2` |
| `thresholds.kill_min_spend_for_cac_kill` | Minimum L7 spend before the CAC-multiplier kill rule applies | `30` |
| `thresholds.kill_zero_conv_spend_gate` | Spend threshold for a zero-conversion ad to be a kill candidate | `15` |
| `thresholds.scale_step_pct` | Budget increase step (and hard cap) per update | `20` |
| `thresholds.cut_step_pct` | Budget decrease step per update | `20` |

Values are in percent except the spend gates, which are in `currency` from
Step 1. Don't invent different defaults than the ones above — these are the
same numbers the reviewer and committer already document, and drifting from
them here would make this skill a second source of truth.

---

## Step 6 — Write the config file

Assemble the full config object (see
`assets/config.example.json` for the exact shape) from Steps 1–5:

```json
{
  "account_name": "<Step 3>",
  "website_id": "<Step 2>",
  "channel": "<Step 2>",
  "ad_account_id": "<Step 1>",
  "currency": "<Step 1>",
  "targets": {
    "target_cac": "<Step 4 or null>",
    "target_roas": "<Step 4 or null>",
    "source": "kickbite | local"
  },
  "thresholds": { "...": "<Step 5>" }
}
```

Show this assembled JSON back to the user as a final visual check before
writing anything.

Write it to `<account_name>.config.json` **in the current working
directory** — never into this skill's own `assets/` folder. That folder
holds the example template shipped with the plugin, not real brand data;
writing a real ad account ID there would mix user data into the plugin
package and risk it landing in version control.

Do not write the file until every field above has a confirmed value. A
partially-filled config is worse than none — it would silently pass
`mba-meta`'s Step 0 check on the next run and proceed with a missing or
wrong value instead of re-triggering onboarding.

---

## Step 7 — Create the log file (only if one doesn't already exist)

Check the same working directory for an existing `log.json`. If there is
none, create it as an empty array `[]`. If one already exists — for
example, another brand's config already lives in this folder and has run
before — **leave it untouched**. `log.json` is shared per working folder
across every brand's config there (entries are disambiguated by their
`account` field), so overwriting it would destroy another brand's run
history.

---

## Step 8 — Hand off

If this skill was invoked because `mba-meta` or
`mba-meta-performance-reviewer` handed off to it in their own Step 0: return
control to that skill now. It should re-resolve the config (it will now
find exactly the one match just written) and continue from its own Step 1 —
don't re-ask the user anything already captured here.

If the user triggered this skill directly: confirm setup is complete and
ask whether they want to run the performance review (`mba-meta`) for this
brand now.

---

## Guardrails

- Never call `ads_update_entity` or `kickbite_set_targets` from this skill —
  it only reads from Meta/Kickbite and writes local JSON files.
- Never guess an `ad_account_id` or `website_id` the user mentions in
  passing without matching it against the lists from Steps 1–2 first.
- Never write `<account_name>.config.json` until every field in Step 6 has
  a confirmed value.
- Never overwrite an existing `log.json` — see Step 7.
- If the user cancels or goes silent mid-flow, don't write any file at all.
- If `ads_get_ad_accounts` or `kickbite_list_markets` errors, stop and
  report rather than treating the failure as "no accounts" or picking a
  cached value from earlier in the conversation.
