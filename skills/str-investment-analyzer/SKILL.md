---
name: str-investment-analyzer
version: 1.1.0
description: Analyze short-term-rental (Airbnb/VRBO) property investments by city. Use whenever someone wants to evaluate buying an Airbnb / short-term rental / STR in a specific market — e.g. "analyze an Airbnb in Wilmington NC for $500k", "is a STR in Gatlinburg a good investment?", "cash-on-cash and hold-period return on a vacation rental".
---

# STR Investment Analyzer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All math, market data, defaults, risk flags, and estimate provenance live **server-side** — this
skill only gathers inputs, calls the tools, and surfaces what they return. It computes and decides
nothing locally; the server is the source of truth.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__forecast_str_market`):
`forecast_str_market`, `list_str_markets`, `analyze_str_property`, `analyze_property_return`.
Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are
written bare for brevity.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

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

## Recommended call sequence

1. (optional) `list_str_markets` → confirm coverage / suggest a city.
2. `forecast_str_market` { location, purchase_price (+ hold_years only if selling) } → one-shot:
   scenarios, risk, self-managed comparison, hold return.
3. (advanced) `analyze_property_return` with `sale_price_override` for a reno/forced-exit scenario.

## Notes
- All decimals are fractions, not percents (20% → `0.20`).
- Surface the server's `disclosures` and `data_provenance` as-is; never present bundled or researched
  data as live quotes — the server already labels origin + `as_of`.
- Not financial advice. Planning estimates only.
