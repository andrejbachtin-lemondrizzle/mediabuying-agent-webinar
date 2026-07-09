---
name: mba-meta-performance-reviewer
description: "Weekly Meta paid ads performance reviewer, driven entirely by a per-brand config file rather than any hardcoded account. Queries the Kickbite Marketing MCP, computes CAC at campaign, adset, and ad level, and renders a three-section action list: budget increase, budget decrease, and ad kills. Trigger whenever the user asks about Meta campaign performance, weekly review, what to increase / decrease / kill, ad CAC, scaling decisions, or budget actions for any client brand — even if they say \"check Meta\", \"how are my ads\", \"performance review\", \"action list\", \"paid review\", or \"what should I scale for [brand]\". Always use this skill for Meta performance work — it enforces the correct aggregate-then-divide CAC formula, the configurable attribution/threshold rules, and per-brand market resolution."
---

# Meta performance reviewer

Produces a three-section action list for a brand's Meta paid social
performance:
1. **Budget increase** — campaigns, adsets, and ads with improving CAC
2. **Budget decrease** — campaigns and adsets with deteriorating CAC
3. **Kill ads** — individual ads burning spend with no return

Data source: **Kickbite Marketing MCP** (`kickbite_marketing_channels_deepdive`)
for performance numbers, plus a read-only pull from the **Meta MCP** for live
entity status (see Step 1) — the only two data sources this skill uses. It
does not read from Google Sheets or query BigQuery directly. Kickbite's `cac`
KPI is already computed server-side as `costs / orders_new`
(aggregate-then-divide, verified against raw campaign-level numbers), and
`change_pct.cac` gives the delta between two date windows directly — this
removes almost all manual arithmetic.

This skill is account-agnostic — it never hardcodes a market, channel, ad
account, or threshold. All of that comes from a brand config file. It also
owns writing this run's entry to the shared `log.json` (see Step 8) —
`mba-meta-commit` updates that same entry later rather than creating its own.

---

## Step 0 — Resolve the brand config

1. Look for a config file (pointed at by the user, sitting in the working
   folder, or already resolved earlier in the conversation by
   `mba-meta`). See `assets/config.example.json` for the shape.
2. If none exists, do not ask an ad-hoc question yourself — invoke the
   `mba-onboarding` skill instead. It resolves the Meta ad account and
   Kickbite market/store together (calling `kickbite_list_markets` as part
   of that), collects targets and thresholds, and writes the config file.
   Wait for it to finish, then re-resolve the config and continue from
   here.
3. Read `website_id` and `channel` from the config (`channel` defaults to
   `"Meta Paid"` if the config omits it — this skill is Meta-specific).
4. Read the four kill/recommendation thresholds from `config.thresholds`.
   If any are missing, use these defaults and say so in the closing
   narrative: `cac_change_pct_threshold: 5`, `kill_cac_multiplier: 2`,
   `kill_min_spend_for_cac_kill: 30`, `kill_zero_conv_spend_gate: 15`.
5. Note `config.account_name` — this is the key used to find this account's
   entries in `log.json` (Steps 2 and 8), so it needs to be consistent across
   runs for the same brand.

Do not proceed to data pulls with a guessed `website_id` — verify it against
`kickbite_list_markets` if there's any ambiguity. Confirming this isn't
optional: this same skill runs across every brand now, so a wrong market ID
silently reviews the wrong client's account.

---

## The key metric: CAC

Use the `cac` KPI directly from `kickbite_marketing_channels_deepdive` — do
not recompute it. It is already `SUM(costs) / SUM(orders_new)` for the
requested window, not a row-by-row average.

**Do not** query day-by-day and average daily `cac` values yourself — this is
a common footgun that produces the wrong number for any multi-day window.

---

## Step 1 — Pull the active-entity registry from Meta

Before any Kickbite calls, fetch the current live state of every
entity in this account directly from Meta. This one pull does two jobs, so
don't fetch it twice: it lets Step 4 filter comparison queries down to
entities that are actually still running (a paused or deleted campaign
shouldn't generate a fresh INCREASE/DECREASE/kill recommendation just
because it still has historical spend), and it gives Step 2 something
concrete to check last run's changes against.

Use `mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_get_ad_entities` (load
via ToolSearch if deferred — see the tool loading note at the bottom), once
per level:

```
ad_account_id: <config.ad_account_id>
level: "campaign"          # then "adset", then "ad" — three calls, same turn
fields: ["id", "name", "status", "daily_budget"]
```

`status` is the field name already confirmed working in `mba-meta-commit`'s
budget-fetch call, so start there rather than guessing `effective_status` —
Meta's own API exposes both, and this MCP wrapper may only normalize to one.
If a call errors on an unexpected field (e.g. `daily_budget` isn't valid at
adset/ad level for this account), drop that field and retry with just
`["id", "name", "status"]` — don't guess field validity per level, let the
API tell you.

From the three responses, build:
- **Active ID sets** per level — entities whose status is *not* one of the
  paused/deleted/archived states. Check the actual values that come back
  before hardcoding a comparison list; Meta's status vocabulary isn't
  identical across campaign/adset/ad (e.g. adsets can inherit a paused state
  from their parent campaign in ways a raw status field won't show).
- **A budget/status lookup by ID** — this is what Step 2 reconciles against.
- A campaign budget-type lookup (CBO vs ABO) — for each campaign, check whether its adsets carry their own daily_budget. If an adset has no daily_budget (null/absent) while its parent campaign does, that campaign is CBO and adset-level budget edits aren't possible for it. If adsets do carry their own daily_budget, the campaign is ABO. Build this as campaign_id → "CBO" | "ABO", not from name-string guessing.

If this call fails, or comes back empty for an account you know has active
spend, stop and report the error rather than treating the failure as "no
active entities" or "everything's active" — either silent default breaks
the point of filtering in Step 4.

---

## Step 2 — Reconcile against the last run

Read the most recent `log.json` entry where `account` matches
`config.account_name`. If there isn't one (first run for this brand), skip
this step entirely — there's nothing to reconcile yet.

For every item in that entry's `budget_increase`, `budget_decrease`, and
`kill_ads` where `result` is `"applied"`, compare it against the live
registry from Step 1:

- **Budget items** — is the entity's current `daily_budget` close to the
  entry's logged `budget_new`? If it's sitting back near the old
  `budget_now` (or anywhere else unexpected), that's drift: something
  outside this workflow changed it back.
- **Kill items** — is the ad's current `status` still paused? If it's
  active again, that's drift too.

Collect every mismatch you find — you'll surface these in Step 7's narrative
*before* discussing this run's own findings, not folded silently into this
run's numbers.

Separately, flag any `"applied"` item whose logged `timestamp` is within the
last few days as **recently changed**. Meta's delivery algorithm tends to
produce noisy performance for a few days after a budget change or a pause
(the learning-phase reset), so if that same entity shows up again in this
run's recommendations, the underlying signal deserves extra skepticism — a
CAC spike two days after a budget increase may just be that, not a new
problem.

---

## Step 3 — Discover the latest date with real spend (T)

Query a short daily breakdown to find the most recent day with nonzero cost —
don't assume "yesterday" is populated; pipeline lag varies by account.

```
Kickbite Marketing:kickbite_marketing_channels_deepdive
  markets: <config.website_id>
  primary_dimension: "daily"
  filters: [{field:"dimension", name:"channel", operator:"is", value:<config.channel>}]
  kpis: ["costs"]
  from_date: <today - 6 days>
  to_date: <today>
```

Call the latest day with `costs > 0` result `T`. All windows below are
relative to `T`.

```
last_7  = [T-6 .. T]
prev_7  = [T-13 .. T-7]
last_3  = [T-2 .. T]
prior_4 = [T-6 .. T-3]
```

---

## Step 4 — Run comparison queries per level

For **each level** (campaign, adset, ad), run **two calls**, both scoped to
`channel = <config.channel>`, both in the **same tool-call turn** so they run
concurrently:

**Call A — WoW (last_7 vs prev_7):**
```
from_date: T-6, to_date: T
compare_from_date: T-13, compare_to_date: T-7
compare_type: "period"
```

**Call B — within-week momentum (last_3 vs prior_4):**
```
from_date: T-2, to_date: T
compare_from_date: T-6, compare_to_date: T-3
compare_type: "period"
```

Shared params for all six calls:
```
markets: <config.website_id>
filters: [{field:"dimension", name:"channel", operator:"is", value:<config.channel>}]
kpis: ["costs", "cac", "orders_new", "roas"]
response_format: "json"
hide_zero: true
limit: 100   (raise if account grows past this many active entities)
```

**Active-only filtering.** Only entities in Step 1's active sets should be
allowed to generate a recommendation — a paused campaign with a scary CAC
number is noise, not signal. Two ways to enforce this; pick one per run and
say which one you used in the Step 7 narrative, since that's useful context
for whoever looks at this next:

1. **Push it into the query** — add a filter
   `{field:"dimension", name:<dimension>, operator:"in", value:[...]}` using
   the active names/IDs from Step 1. Try this first at the **ad level**
   using `sub_sub_campaign_id` (IDs, not names — ad names aren't guaranteed
   unique, see the known limitation below). At the campaign/adset level,
   Kickbite's dimension is name-based, so use active *names* there, accepting
   the small residual risk of a false match if an active and inactive entity
   happen to share a name.
2. **Pull unfiltered, drop after** — if the `in` filter behaves badly with a
   large list (silently truncated, or the API rejects lists past some size),
   fall back to pulling the comparison data unfiltered as before and drop
   any row whose name/ID isn't in Step 1's active set before computing
   recommendations.

Neither approach has been exercised yet against a real large account — the
first live run should confirm which one Kickbite actually handles cleanly,
and that finding should get folded back into this skill.

Dimension mapping (confirmed against live data):

| Level    | `primary_dimension`    |
|----------|-------------------------|
| Campaign | `campaign`              |
| Adset    | `sub_campaign`          |
| Ad       | `sub_sub_campaign`      |

For the **adset** calls, add `secondary_dimension: "campaign"` so each row
carries its parent campaign name (returned in the `secondary` field).

For the **ad** calls, add `secondary_dimension: "campaign"` as well — this
gives Ad + parent Campaign, at the cost of not also showing the parent Adset
in the same call. If you need the Adset column too, that's a second ad-level
call merged by ad name/ID, roughly doubling ad-level calls.

**Known limitation:** `sub_sub_campaign` (ad name) is not guaranteed unique
account-wide. If two ads in different campaigns share a name, cross-check
`sub_sub_campaign_id` before making a kill decision on ambiguous rows.

**Account benchmark** (for the kill threshold) — one more call, no
dimension breakdown:
```
markets: <config.website_id>
primary_dimension: "channel"
filters: [{field:"dimension", name:"channel", operator:"is", value:<config.channel>}]
kpis: ["costs", "cac"]
from_date: T-6, to_date: T
response_format: "json"
```
This single row's `cac` is `account_l7_cac`.

---

## Step 5 — Read recommendations directly from the API response

Each row already contains:
- `kpis.cac` — CAC for the requested window
- `kpis.costs`, `kpis.orders_new`, `kpis.roas`
- `change_pct.cac` — % change vs the comparison window (positive = CAC
  went up = worse; negative = CAC went down = better)

If `change_pct` is missing entirely for a row, treat it as a **new entity**
(no comparable prior-period data) — do not compute a percentage from zero.

### Recommendation logic (apply in priority order)

Let `T = config.thresholds.cac_change_pct_threshold` (default `5`).

Adset-level gate: before applying the table below, check campaign_budget_type for the adset's parent campaign (from Step 1). If it's CBO, do not emit an adset-level INCREASE/DECREASE row — that adset has no independently-editable budget. Fold its signal into the parent campaign's row instead (the campaign-level recommendation already covers it, since campaign is where the CBO budget lever actually lives).

| Condition (from Call B, within-week) | Recommendation |
|---|---|
| `change_pct.cac` present AND ≤ -T | **INCREASE** |
| `change_pct.cac` present AND ≥ +T | **DECREASE** |
| Within-week `change_pct.cac` missing — fallback to Call A (WoW): ≤ -T | **INCREASE** |
| Within-week `change_pct.cac` missing — fallback to Call A (WoW): ≥ +T | **DECREASE** |
| Everything else (within ±T%, no data, brand-new entity) | **HOLD** |

When Call A (WoW) and Call B (within-week) disagree in direction, flag this
explicitly in the post-widget narrative. Don't resolve the conflict
silently; surface it.

### Kill logic (ad level only)

Let `M = config.thresholds.kill_cac_multiplier` (default `1.5`),
`S = config.thresholds.kill_min_spend_for_cac_kill` (default 150`),
`Z = config.thresholds.kill_zero_conv_spend_gate` (default `150`).

Kill an ad if either is true, using **Call A (last_7)** figures:
- `kpis.cac > M × account_l7_cac` AND `kpis.costs > S`
- `kpis.orders_new == 0` AND `kpis.costs > Z`

Note campaign type from the campaign name string (rows where `secondary`
starts with `"ASC"` vs `"CBO"`) — ASC kills are lower priority (Meta's
algorithm partly manages ad-level allocation within ASC); CBO kills are
high priority. This naming convention is Strayz's; if a brand's campaigns
don't follow it, skip this prioritization nuance and treat all kills
equally.

---

## Step 6 — Render the action list widget

Call `mcp__visualize__read_me` with `modules: ["data_viz"]` once per session,
then build with `mcp__visualize__show_widget`. Format amounts using
`config.currency`: € for EUR, $ for USD, £ for GBP; otherwise show the
3-letter code after the amount (e.g. `120 SEK`).

### Header: three metric cards
- Account CAC last 7d (from the account benchmark call, integer, in
  config.currency)
- Total spend last 7d (comma-formatted, in config.currency)
- WoW CAC change (%, green if negative = improving, red if positive = worsening)

### Section 1 — Budget increase
Table: Level pill (Campaign / Adset / Ad) · Name · Parent · L7 CAC · Signal

- Campaigns with INCREASE
- Adsets with INCREASE — only if campaign_budget_type == ABO. For CBO adsets, don't add a row; the campaign-level row above already covers it.

### Section 2 — Budget decrease
Table: Level · Name · Parent (Campaign) · L7 CAC · Signal

- Campaigns with DECREASE
- Adsets with DECREASE — only if campaign_budget_type == ABO. For CBO adsets, don't add a row; the campaign-level row already covers it.
- Signal column: show the WoW `change_pct.cac` so the reader sees magnitude

### Section 3 — Kill ads
Table: Ad · Campaign · L7 Spend · L7 CAC · Reason

(No Adset column here — see the Step 4 trade-off note.)
Sort by L7 Spend descending. Reason: "CAC {currency}X, +Y% WoW" or
"{currency}Z spend · 0 conv" — under 6 words.

### Design rules
- CSS variables for all colors (light/dark mode compatible)
- Round CAC to nearest integer, percentages to nearest integer
- Green: `var(--color-background-success)` / `var(--color-text-success)`
- Red: `var(--color-background-danger)` / `var(--color-text-danger)`
- New entities (no comparison data): small info-colored "new" badge
- Recently-changed entities (flagged in Step 2): small caution badge or
  footnote marker, distinct from the "new" badge
- Footer: one line noting `config.account_name`, the attribution model in
  use (check `kickbite_list_kpis` / account settings for which one is
  active — don't assume "AI Click & View"), and the data date range. If any
  threshold fell back to a default because the config omitted it, say so
  here too.

---

## Step 7 — Post-widget narrative (3–5 sentences)

Lead with anything from Step 2 before discussing this run's own findings:

- **Drift since last run** — any `"applied"` change from the previous log
  entry that no longer matches Meta's live state (a reverted budget, a
  killed ad that's active again). Name the entity and what changed back.
  If Step 2 found nothing, skip this bullet rather than stating a negative.
- **Recently-changed caution flags** — entities logged as applied within the
  last few days that reappear in this run's recommendations; note the
  learning-phase caveat rather than treating the signal at face value.

Then cover this run's own findings:
- Main driver of the WoW CAC change (which campaign moved the needle most)
- Most urgent kill (highest spend wasted)
- Any conflicted signals (within-week vs WoW pointing opposite directions)
- Any adsets/ads that disappeared mid-week (present in Call A's comparison
  period but absent or zero-cost in the current window — likely paused,
  which Step 1's registry can confirm directly instead of guessing)

---

## Step 8 — Write the log entry

After the narrative, append one new entry to `log.json` (in the same working
folder as the config file; create the file as an empty array first if it
doesn't exist yet). This entry records what was *reviewed* — nothing has
been executed yet, so every entity gets `"result": null`. `mba-meta-commit`
finds and updates this exact entry later; it does not get replaced or
duplicated.

Kickbite's dimension data doesn't carry an adset ID (only the adset name),
but Step 1's Meta registry pull does — it fetches real adset IDs by name.
When assembling each budget_increase/budget_decrease item below, backfill
`adset_id` from that Step 1 lookup by matching `adset_name`; only leave it
`null` if the name genuinely isn't found in the registry (e.g. a brand-new
adset Meta hasn't synced status for yet).

```json
{
  "timestamp": "<run time, ISO 8601>",
  "account": "<config.account_name>",
  "ad_account_id": "<config.ad_account_id>",
  "period": { "last_7": "...", "prev_7": "...", "last_3": "...", "prior_4": "..." },
  "account_cac_l7": 0,
  "account_cac_wow_change_pct": 0,
  "status": "reviewed_not_executed | reviewed_pending_confirmation",
  "budget_increase": [
    {
      "level": "campaign | adset",
      "campaign_id": "...", "campaign_name": "...",
      "adset_id": null, "adset_name": null,
      "ad_id": null, "ad_name": null,
      "l7_cac": 0, "signal": "...",
      "budget_now": 0, "budget_new": 0,
      "result": null, "error": null
    }
  ],
  "budget_decrease": [ "...same shape as above..." ],
  "kill_ads": [
    {
      "campaign_id": "...", "campaign_name": "...",
      "adset_id": null, "adset_name": null,
      "ad_id": "...", "ad_name": "...",
      "l7_spend": 0, "l7_cac": 0, "reason": "...",
      "result": null, "error": null
    }
  ],
  "thresholds_used": { "cac_change_pct_threshold": 5, "kill_cac_multiplier": 2, "kill_min_spend_for_cac_kill": 30, "kill_zero_conv_spend_gate": 15, "scale_step_pct": 20, "cut_step_pct": 20 },
  "notes": null
}
```

Set the top-level `status`:
- All three sections empty → `reviewed_not_executed`
- Any section non-empty → `reviewed_pending_confirmation` (commit hasn't run
  yet — it may never run, if the user doesn't act on this review)

Use `notes` for anything that doesn't fit a structured field: a threshold
that fell back to its default, an `adset_id` that couldn't be resolved from
the Step 1 registry, the active-filter approach used in Step 4, or a
Step 1/2 data-quality caveat. Leave it `null` if there's nothing to add —
don't pad it with restated information that's already structured elsewhere.

Keep this run's `timestamp` in context after writing — `mba-meta-commit`
matches on `account` + `timestamp` to find and update this exact entry
rather than creating a new one, however the handoff to it happens (directly,
or via `mba-meta`).

---

## Kickbite Marketing MCP reference

Tool: `Kickbite Marketing:kickbite_marketing_channels_deepdive`

| Param | Notes |
|---|---|
| `markets` | Required. Comma-separated website_ids — comes from `config.website_id`. |
| `primary_dimension` | `campaign` \| `sub_campaign` (=adset) \| `sub_sub_campaign` (=ad) \| `sub_sub_campaign_id` (unique but unreadable) |
| `secondary_dimension` | Adds one parent/cross-tab column, e.g. `campaign` |
| `filters` | `[{field:"dimension"\|"kpi", name, operator, value}]`. Operators: `is, not, in, not_in, gte, lte, between, contains, not_contains, starts, ends, first, last, pareto` — no `gt`/`lt`/`eq`. |
| `kpis` | Valid set (channels view) includes: `costs, cac, orders_new, roas, revenue, aov, cpo, sessions, ...` — call `kickbite_list_kpis` if unsure; the "deepdive" view KPI list is different from "channels" view and will 400 if mixed up. |
| `from_date`/`to_date` | YYYY-MM-DD, defaults server-side to yesterday if omitted — don't rely on the default, pass explicit dates from Step 3. |
| `compare_from_date`/`compare_to_date`/`compare_type` | Enables `change_pct` / `change_abs` per KPI, computed correctly (not a naive average). |
| `response_format` | Use `"json"` for programmatic parsing — markdown mode is for direct human display only. |

Helper tools:
- `kickbite_list_markets` — resolve account name → website_id
- `kickbite_list_kpis` — list valid KPI slugs per market/view (avoid guessing slugs)
- `kickbite_list_dimension_values` — enumerate exact dimension values before filtering

---

## Meta MCP tool loading note

`mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_get_ad_entities` is deferred
— load it via `ToolSearch` with
`select:mcp__3ebf67ff-2ae5-443f-a637-25bdede46a7a__ads_get_ad_entities`
before Step 1. This skill only ever reads through this tool; it never calls
`ads_update_entity` — all writes belong to `mba-meta-commit`.
