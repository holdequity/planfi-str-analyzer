---
name: str-investment-analyzer
version: 1.1.2
description: Analyze short-term-rental (Airbnb/VRBO) property investments by city. Use whenever someone wants to evaluate buying an Airbnb / short-term rental / STR in a specific market — e.g. "analyze an Airbnb in Wilmington NC for $500k", "is a STR in Gatlinburg a good investment?", "cash-on-cash and hold-period return on a vacation rental".
---

# STR Investment Analyzer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All math, market data, defaults, risk flags, and estimate provenance live **server-side** — this
skill only gathers inputs, calls the tools, and surfaces what they return. It computes and decides
nothing locally; the server is the source of truth.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__forecast_str_market`):
`forecast_str_market`, `list_str_markets`, `analyze_str_property`, `analyze_property_return`,
`analyze_str_tax_loophole`.
Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are
written bare for brevity.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — Gather inputs (just two are required)

1. **location** — city + state, e.g. `"Wilmington, NC"` (required).
2. **purchase_price** — dollars, e.g. `500000` (required).

**Hold horizon:** ask if it comes up naturally, but you don't have to manage it. Pass `hold_years`
if the user has a sale horizon; **omit it for an indefinite buy-and-hold**. The server then returns
`hold_horizon.mode` (`fixed` / `indefinite`) and, when indefinite, labels the sale-based return
`illustrative` with a note — just surface what it returns (see Step 4). No defaults to track here.

Everything else is optional — only pass a value when the user volunteers one (the server supplies and
labels every default): `nightly_rate`, `occupancy_rate`, `annual_appreciation`, `down_payment_percent`,
`closing_costs_percent`, `mortgage_rate`, `mortgage_term_years`, `furniture_cost`, `cleaning_fee`,
`monthly_property_tax`, `monthly_insurance`, `monthly_hoa`, `monthly_utilities`, `management_fee_percent`,
`platform_fee_percent`, `annual_maintenance_percent`, `index_return`.

> "What markets do you cover?" → call **`list_str_markets`** (full catalog by category, total count,
> asOf range). Use it to confirm coverage or suggest cities.

## Step 2 — Run the one-shot forecast (primary path)

Call **`forecast_str_market`** with the location + price. It auto-populates nightly rate & occupancy
from the bundled dataset and returns, in a single call: the cash-flow scenarios
(pessimistic/realistic/optimistic), break-even occupancy, a top-level `risk` flag, a
`self_managed_comparison`, `data_provenance`, and the hold-period return.

```
forecast_str_market({
  location: "Wilmington, NC",
  purchase_price: 500000
  // hold_years: pass only if the user has a sale horizon; omit for buy-and-hold
  // any other field: pass only if the user specifies it
})
```

### If the market isn't covered (`needs_market_data: true`)
`forecast_str_market` returns a soft `needs_market_data` envelope with `suggestions` (nearest covered
markets) and a `coverage_note`. **Do not proceed silently.** Tell the user their market isn't in the
dataset, show the suggestions, then ask whether to: (a) use a suggested nearby covered market, or
(b) web-research the current Airbnb/VRBO ADR + occupancy (AirDNA, AirROI, AirBtics, Rabbu) and re-call
with explicit `nightly_rate` + `occupancy_rate` (the server labels them `confidence: "researched"`).
Never invent numbers or pick a nearby market without confirming.

## Step 3 — Optional finer control

- **Self-managed comparison:** already included in every `forecast_str_market` response as
  `self_managed_comparison` — surface it; no second call needed.
- **Reno / forced exit value:** call **`analyze_property_return`** with the realistic `annual_cash_flow`
  as `annual_net_cash_flow`, plus `sale_price_override` for an explicit exit price (e.g. after-reno ARV).
- **Cash-flow only / custom STR expenses:** `analyze_str_property` accepts `location` (auto-fill) or
  manual `nightly_rate`/`occupancy_rate` and the full STR expense set for tuning.

## Step 4 — Present what the server returned

Surface the server's fields as-is — do not relabel, recompute, or add your own thresholds:

- **`assumed_defaults[]`** — every analysis tool (`forecast_str_market`, `analyze_str_property`,
  `analyze_property_return`) returns a structured `assumed_defaults` array of
  `{ field, assumed_value, note }` for each forecast-driving input left at its server default
  (e.g. `down_payment_percent`, `mortgage_rate`, `management_fee_percent`, and — when you omit
  `hold_years` — the illustrative `hold_years`). Read these back to the user so they can confirm
  or override them; they are NOT the same as the static prose in `disclosures.key_assumptions`,
  and `disclosures.not_advice` is a boolean flag, not a message.
- **`data_provenance`** — `origin`, `nightly_rate`, `occupancy_rate`, `source`, `source_url` (cite it),
  `as_of`, `confidence`. Present where the numbers came from exactly as returned. Read-rule: lead with
  the lower of `data_provenance.confidence` (match quality) and `disclosures.confidence` (forecast
  confidence) when they diverge.
- **`risk`** `{ level, reasons[] }` — the server's top-level risk flag. If `level` is elevated, lead
  with it and list the `reasons` up front — do not bury it.
- **`self_managed_comparison`** — present the managed-vs-self delta the server computed.
- **`str.realistic`** (+ pessimistic/optimistic) — `annual_cash_flow`, `monthly_cash_flow`,
  `cash_on_cash_return_pct`, `cap_rate_pct`, `break_even_occupancy_pct`. Lead with realistic
  cash-on-cash, cap rate, and break-even occupancy (vs the market's actual occupancy).
- **`hold_horizon`** + **`hold_return`** — if `hold_horizon.mode` is `fixed`, feature the
  annualized/total return. If `indefinite` (or `hold_return.illustrative` is true), lead with the
  recurring economics and present the sale return only as the server's labeled hypothetical.
- **`hold_return.opportunity_cost`** (always present) — frame as: "instead of buying the STR for N
  years, park the SAME cash in a ~7% index fund for the SAME N years — which wins?" Present
  `index_terminal` vs `str_terminal`, `winner`, `delta`, both multiples, and `breakeven_index_return`
  if non-null. Always carry the server's `note` verbatim (it states the comparison is NOT
  risk-adjusted).

### intent → analyze_str_tax_loophole

Real-user phrasings that hit this: _"can I write off my Airbnb against my W-2?"_, _"the STR
loophole / short-term rental tax loophole"_, _"non-passive rental losses without being a
real-estate professional"_, _"cost-seg / bonus depreciation to offset my salary"_, _"average guest
stay under 7 days material participation"_, _"how much tax does the Airbnb loophole actually save
me?"_, _"will depreciation recapture bite me when I sell my short-term rental?"_

**Always CALL `analyze_str_tax_loophole` for these — do not answer from general knowledge or quote
rules of thumb from memory.** When the user gives the numbers (average guest stay, owner hours / who
else logs hours, year-1 loss, W-2 income), run it and lead with its real output: the average-stay
classification (non-passive vs passive), the material-participation determination, the allowable
first-year loss against W-2, the ordinary tax saved, and the depreciation-recapture liability
projected at sale.

The §469 STR "loophole" is: when **average guest stay ≤ 7 days** (or ≤ 30 with substantial
services) the activity is **not a rental activity** under Treas. Reg. §1.469-1T(e)(3), so with
**material participation** (the 100-hour-and-more-than-anyone-else test or the 500-hour test) the
losses are **non-passive** — deductible against W-2/active income **without** 750-hour real-estate-
professional status, and with **no $25k special-allowance haircut**.

Wire params (snake-case): `average_stay_days`, `substantial_services`, `material_participation_hours`,
`hours_anyone_else`, `year_1_loss`, `ordinary_taxable_income` (or `w2_income`), `filing_status`,
`magi`, `state_marginal_rate`, `section_1250_accumulated_depreciation`, `cost_seg_1245_basis`,
`projected_total_gain_at_sale`, `tax_year`, optional `plan_id` (+ overrides). `filing_status`,
`ordinary_taxable_income`, and `magi` are plan-derivable — omit them when a `plan_id` is in scope and
the server fills them from plan scalars.

**If material participation FAILS**, the loss is passive and suspended — the tool returns the
suspended amount and routes you to **`analyze_passive_losses`** (the $25k special-allowance path).
Follow that hand-off; don't quote the passive-loss rules from memory either.

## Recommended call sequence

1. (optional) `list_str_markets` → confirm coverage / suggest a city.
2. `forecast_str_market` { location, purchase_price (+ hold_years only if selling) } → one-shot:
   scenarios, risk, self-managed comparison, hold return.
3. (advanced) `analyze_property_return` with `sale_price_override` for a reno/forced-exit scenario.

## Related skills
- **long-term-rental-analyzer** — the buy-and-hold / long-term-rental complement to this nightly-STR
  skill: year-1 Schedule E P&L, 27.5-yr depreciation shelter, §1250 recapture, NIIT, and 1031
  deferral (the after-tax layer on a non-nightly rental).
- **rent-vs-buy** — owning a primary residence vs renting + investing the difference.

## Notes
- All decimals are fractions, not percents (20% → `0.20`).
- Surface the server's `disclosures` and `data_provenance` as-is; never present bundled or researched
  data as live quotes — the server already labels origin + `as_of`.
- Not financial advice. Planning estimates only.
