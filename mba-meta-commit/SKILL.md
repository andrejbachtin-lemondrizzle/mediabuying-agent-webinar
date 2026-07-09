---
name: mba-meta-commit
description: >
  Executes Meta ads changes for any client brand after a performance review,
  driven entirely by a per-brand config file (ad account ID, currency, and
  budget step sizes) rather than a hardcoded account. Trigger this skill
  whenever the user wants to apply, upload, execute, push, or confirm Meta ad
  changes — even if they say "go ahead", "do it", "accept", "execute the
  changes", "apply the recommendations", "update the budgets", "kill those
  ads", "push to Meta", or "upload the changes". Also trigger when the user
  accepts or approves an action list produced by mba-meta-performance-reviewer.
  The skill builds a formatted confirmation table (Type | Name | Budget now |
  Budget new | Kill), waits for explicit user sign-off, then executes via
  Meta MCP: budget changes via ads_update_entity, ad pauses via
  ads_update_entity with status PAUSED. Always use this skill when the intent
  is to make live changes to a Meta ad account — do not execute ad changes
  without it.
---

# Meta Ads Uploader

Converts a `mba-meta-performance-reviewer` output into a confirmed, executed set
of changes on a brand's Meta ad account. The ad account ID, currency, and
budget step percentages all come from that brand's config file — never
hardcode them.

This skill never creates its own `log.json` entry. `mba-meta-performance-reviewer`
already wrote one for this run (its Step 8); this skill's whole job here is
to find that entry and update it with what actually happened — see Step 1.

---

## Step 0 — Resolve the brand config

Look for the config file already established in this conversation (by
`mba-meta` or `mba-meta-performance-reviewer`), or read one directly if the
user points at it. See `assets/config.example.json` for the shape. You need:

- `ad_account_id` — the Meta ad account to write to
- `currency` — for display formatting
- `thresholds.scale_step_pct` / `thresholds.cut_step_pct` — budget step
  sizes (both default to `20` if the config omits them)
- `account_name` — used to find the right entry in `log.json` in Step 1

Never guess `ad_account_id` — writing a budget change to the wrong ad
account is a real-money mistake, not a cosmetic one. If it's missing or
ambiguous, stop and ask rather than reusing the last account you touched.

---

## Step 1 — Locate the log entry to update

Find the entry `mba-meta-performance-reviewer` wrote for this run in
`log.json` (same working folder as the config file), matched on
`account == config.account_name` and the `timestamp` carried forward from
the reviewer — however that handoff happened (directly, or relayed through
`mba-meta`).

If no matching entry exists — for example, the user pasted in review output
from outside this conversation, or ran the reviewer against a different log
file — stop and ask rather than guessing which entry to update or silently
creating a new one. A stray second entry for the same run is exactly the
kind of log inconsistency this whole update exists to prevent.

Keep a reference to this entry open for the rest of the run: Step 6 writes
`declined` into it on a "no", and Step 7 writes per-entity results into it
after execution. There is only ever one entry per run, updated in place.

---

## Step 2 — Check for performance review data

Look in the current conversation for output from `mba-meta-performance-reviewer`. You need:
- A list of campaigns/adsets with INCREASE or DECREASE recommendations
- A kill list of individual ads (with their Ad IDs)
- The account-level L7 CAC (used for kill threshold verification)

If this data is not in context, run the `mba-meta-performance-reviewer` skill
first — with the same config — and wait for it to complete before continuing.

---

## Step 3 — Fetch current budgets

For every INCREASE or DECREASE entity, you need the current daily budget from Meta.

Use `mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_get_ad_entities` (load via ToolSearch if deferred):
- `ad_account_id`: `<config.ad_account_id>`
- `level`: `"campaign"`
- `fields`: `["id", "name", "daily_budget", "status"]`

**Budget structure note:** Strayz holds budget at the campaign level for all
active campaign types (ASC and CBO), with adsets carrying no individual
budget — so budget changes always go to the campaign even when the
recommendation originated from an adset signal. A different brand's account
may be structured differently (e.g. budget held at adset level); if the
fetched entities don't match this pattern, confirm the right level with the
user before proceeding rather than assuming Strayz's structure is universal.
If the same campaign appears multiple times (e.g., from two adset signals),
consolidate into one row.

---

## Step 4 — Calculate new budgets

**INCREASE:** `new_budget = round(current_budget * (1 + scale_step_pct/100))` — hard cap at `+scale_step_pct%`, never exceed.

**DECREASE:** `new_budget = round(current_budget * (1 - cut_step_pct/100))`.

`scale_step_pct` and `cut_step_pct` come from `config.thresholds` (default
`20` each if the config omits them).

The Meta API expects budgets in **cents** (minor currency units) regardless
of `config.currency`. When calling `ads_update_entity`, multiply the amount
by 100. Example: with a 20% step, €180/day → `18000`.

---

## Step 5 — Build the action table

Present the full list before executing anything. Format amounts using
`config.currency`: € for EUR, $ for USD, £ for GBP, otherwise the 3-letter
code.

```
Type       | Name                                   | Budget now | Budget new | Kill
-----------|----------------------------------------|------------|------------|-----
Campaign   | ASC - Katze - 2026 – New               | €150/day   | €180/day   | —
Campaign   | ASC - Katze - 2026 – Old               | €150/day   | €180/day   | —
Ad         | Founder Ads | Der Unterschied | 1       | —          | —          | YES
Ad         | Founder Ads | Der Unterschied | 1–Kopie | —          | —          | YES
```

For ads in the kill list, include the parent adset and campaign in a note
below the table or in a second column so it's unambiguous which ad is
targeted (ad names repeat across campaigns).

Add a one-line summary at the end: **"X budget changes · Y ad kills"**

Then ask: **"Confirm to execute? (yes / no / modify)"**

**Do not call any Meta API until the user replies with an explicit confirmation.**

---

## Step 6 — Handle the user's response

**"yes" / "go ahead" / "do it" / "confirm"** → proceed to Step 7.

**"no" / "cancel"** → update every entity in the Step 1 log entry to
`"result": "declined"`, set the entry's top-level `status` to `"declined"`,
and stop. Confirm clearly: "No changes were made." Right now a decline
produces no log update at all, which leaves the entry stuck at
`reviewed_pending_confirmation` forever and misrepresents what actually
happened — don't skip this.

**"modify" or partial acceptance** → ask what to change (e.g. "skip the CBO increase", "only kill the first 3 ads"). Update the table, show it again, and ask for confirmation. Repeat until you get a clean yes or no. Any entity dropped this way gets `"result": "declined"` in Step 7, same as a full decline — it was reviewed but the user chose not to act on it, which is a distinct outcome from a failed API call.

**Partial list** (e.g. "just do the kills") → execute only the confirmed subset in Step 7; every entity outside that subset also gets `"result": "declined"` there, not left `null` — `null` should only ever mean "never made it to a confirmation gate."

---

## Step 7 — Execute (kills first, then budgets)

Kills always go before budget increases. Reason: you don't want to scale spend on a campaign that still has a live high-CAC ad consuming budget.

### Kill ads (pause)

For each ad in the kill list, call `mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_update_entity` (load via ToolSearch):

```json
{
  "ad_account_id": "<config.ad_account_id>",
  "entity_id": "<Ad_ID>",
  "entity_type": "ad",
  "fields": "{\"status\": \"PAUSED\"}"
}
```

Use the `Ad_ID` values from the performance review data (the numeric IDs, not the names).

### Budget changes

For each campaign with a new budget, call `mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_update_entity`:

```json
{
  "ad_account_id": "<config.ad_account_id>",
  "entity_id": "<Campaign_ID>",
  "entity_type": "campaign",
  "fields": "{\"daily_budget\": <new_budget_in_cents>}"
}
```

You can batch kills in parallel (they're independent), then batch budget changes in parallel after kills complete.

### Write the result back into the log entry as each call completes

Don't wait until everything finishes to record outcomes in one blanket pass
— write each entity's result into the matching item in the Step 1 log entry
as soon as that call returns, so a batch of 11 where 10 succeed and 1 fails
ends up as 10 `"applied"` and 1 `"failed"`, not one status covering all 11:

- Call succeeds → `"result": "applied"`
- Call fails → `"result": "failed"`, `"error": "<short reason from the API error>"`

Once every entity that reached this step has a non-null `result`, derive the
entry's top-level `status`:

| Condition | `status` |
|---|---|
| Every entity `"applied"` | `executed` |
| Mix of `"applied"`, `"failed"`, and/or `"declined"` (partial acceptance) | `partially_executed` |
| Every entity `"declined"` | `declined` (this is also set directly in Step 6 on a full "no", before execution ever runs) |

If any API call fails, report the error inline and continue with the rest. Never silently skip a failure — name the entity that failed and the error reason, both in the completion report (Step 8) and in the log entry's `error` field.

---

## Step 8 — Completion report

After all calls finish:

```
Done.

Killed (paused):
  ✓ Founder Ads | Der Unterschied | 1  (ad 6946141803105)
  ✓ Founder Ads | Der Unterschied | 1 – Kopie  (ad 52512380045509)
  ...

Budget changes:
  ✓ ASC - Katze - 2026 – New  €150 → €180/day
  ✓ ASC - Katze - 2026 – Old  €150 → €180/day
  ...

X of Y changes applied successfully.
```

If any API call fails, report the error inline and continue with the rest. Never silently skip a failure — name the entity that failed and the error reason.

The log entry is already up to date by this point (Step 7) — this report is
for the user in chat, not a second place that needs writing.

---

## Key constants (all from config — nothing here is hardcoded)

| Field | Source |
|-------|--------|
| Ad account ID | `config.ad_account_id` |
| Currency | `config.currency` (budgets are always sent to Meta in cents regardless of display currency) |
| Max budget increase | `config.thresholds.scale_step_pct` per update (default 20%) |
| Default budget decrease | `config.thresholds.cut_step_pct` per update (default 20%) |
| Performance review skill | `mba-meta-performance-reviewer` |
| Log file | `log.json`, same working folder as the config file |

---

## Tool loading note

The Meta MCP tools (`ads_get_ad_entities`, `ads_update_entity`) are deferred — load them via `ToolSearch` with `select:mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_update_entity,mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_get_ad_entities` before calling them.
