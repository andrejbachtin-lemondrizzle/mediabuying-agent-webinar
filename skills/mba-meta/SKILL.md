---
name: mba-meta
description: >
  Full Meta ads management cycle for any client brand: runs the performance
  review and, if there are actions to take, walks through the upload flow —
  with a mandatory confirmation step before any change goes live. Driven
  entirely by a per-brand config file, so the same skill works across every
  Meta ad account Kickbite manages, not just one.

  Trigger whenever the user wants to do a complete Meta review-and-act cycle,
  even if they phrase it as "run the Meta workflow", "check and apply Meta
  changes", "do the weekly Meta thing", "manage my Meta ads", "review and
  update Meta", or "handle Meta for [brand]". Also trigger when the user wants
  to see performance AND is open to acting on it in the same session. Never
  skip the confirmation step — no changes are made without the user
  explicitly approving the action table first.
---

# Meta Manager

End-to-end orchestrator that resolves which brand you're working on, runs the
performance review, evaluates whether any action is needed, and (if so) hands
off to the uploader — which always pauses for explicit confirmation before
touching anything live.

This skill is account-agnostic: every brand-specific number (market,
attribution channel, ad account ID, currency, and the thresholds that decide
increase/decrease/kill) lives in a config file, not in these instructions.
That's what makes it possible to run this for Strayz today and a different
brand tomorrow without editing any of the three Meta skills.

---

## Step 0 — Resolve the brand config

Look for a config file before doing anything else:

Search, in this order, for a brand config (a JSON file with account_name, ad_account_id, website_id fields — not the file literally named config.example.json):

A path the user explicitly names.
Any file matching *.config.json in the current working directory.
Any file matching *.config.json under this skill's own assets/ folder, excluding config.example.json.
If step 3 turns up exactly one match, treat it as the active brand and proceed — do not ask for confirmation, since there's no ambiguity to resolve. If step 3 turns up more than one match, stop and ask the user which brand they mean — never guess based on recency.

If step 3 turns up zero matches, no brand is configured yet. Do not ask an ad-hoc question yourself — invoke the `mba-onboarding` skill instead. It resolves the Meta ad account and Kickbite market/store together, collects targets and thresholds, and writes the config file. Wait for it to finish, then re-resolve the config (it will now find exactly one match) and continue to Step 1.

Config fields:

| Field | Meaning |
|---|---|
| `account_name` | Human-readable label, used in narrative text |
| `website_id` | Kickbite market ID passed to `kickbite_marketing_channels_deepdive` |
| `channel` | Paid channel filter value, e.g. `"Meta Paid"` |
| `ad_account_id` | Meta ad account ID for the uploader's write calls |
| `currency` | 3-letter code; used to pick a display symbol (€/$/£) for amounts |
| `thresholds.cac_change_pct_threshold` | % CAC move (within-week or WoW) that triggers INCREASE/DECREASE instead of HOLD. Strayz default: `5` |
| `thresholds.kill_cac_multiplier` | An ad's CAC above `kill_cac_multiplier × account L7 CAC` is a kill candidate. Strayz default: `2` |
| `thresholds.kill_min_spend_for_cac_kill` | Minimum L7 spend (in `currency`) before the CAC-multiplier kill rule applies, so low-spend noise doesn't get killed. Strayz default: `30` |
| `thresholds.kill_zero_conv_spend_gate` | An ad with 0 conversions and spend above this is a kill candidate regardless of CAC. Strayz default: `15` |
| `thresholds.scale_step_pct` | Budget increase step, also the hard cap per update. Strayz default: `20` |
| `thresholds.cut_step_pct` | Budget decrease step. Strayz default: `20` |

If a threshold is missing from the config, `mba-meta-performance-reviewer` and
`mba-meta-commit` fall back to the Strayz defaults noted above and say so
explicitly in their output — never invent a different number silently.

Pass the resolved config (or its file path) forward when you invoke the two
subskills below, so they don't have to re-resolve it themselves.

---

## Step 1 — Run the performance review

Invoke the `mba-meta-performance-reviewer` skill in full, with the brand config
from Step 0. This queries the Kickbite Marketing MCP, computes CAC at
campaign / adset / ad level, and produces a three-section action list
(Budget increase · Budget decrease · Kill ads) plus the post-widget
narrative.

Wait for the review to complete before continuing.

---

## Step 2 — Assess whether action is needed

After the review renders, check whether any of the three sections contain
at least one item:

- **Section 1 (Budget increase)**: any INCREASE row?
- **Section 2 (Budget decrease)**: any DECREASE row?
- **Section 3 (Kill ads)**: any ad in the kill list?

**If none of the three sections have items** — the account is in good shape.
Respond with a short confirmation, e.g.:

> "Review complete — no changes needed. All campaigns are within the
> configured CAC threshold and no ads hit the kill criteria."

Stop here. Do not invoke the uploader.

---

## Step 3 — Hand off to the uploader (only if actions exist)

If there is at least one actionable item from Step 2, invoke the
`mba-meta-commit` skill, passing it the brand config and the performance
review data already in context.

The uploader will:
1. Fetch current budgets from Meta (`ad_account_id` from config)
2. Compute new budget values (± `scale_step_pct` / `cut_step_pct` from config)
3. **Build and display a confirmation table** (Type | Name | Budget now | Budget new | Kill)
4. Ask: "Confirm to execute? (yes / no / modify)"
5. **Wait for the user's explicit reply** before making any API call

This confirmation step is mandatory and cannot be skipped. If the user says
"no" or "cancel", the uploader stops and no changes are made. If the user
modifies the list, the uploader shows the updated table and asks again.

---

## Guardrails

- Never call `ads_update_entity` or any Meta write tool directly from this
  skill. All writes go through the uploader, which owns the confirmation gate.
- If the performance review errors out (e.g. a Kickbite MCP timeout), stop
  and report the error. Do not proceed to the upload step on incomplete data.
- If the user interrupts mid-flow and asks to "just do the kills" or "skip
  the decreases", relay the partial scope to the uploader rather than
  handling it here.
- If more than one brand config could plausibly match what the user typed
  (e.g. they say "check Meta" and there are three configs in the working
  folder), ask which one rather than picking the most recently used.
