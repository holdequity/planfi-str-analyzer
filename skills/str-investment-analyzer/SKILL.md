---
name: str-investment-analyzer
version: 1.0.1
description: Analyze short-term-rental (Airbnb/VRBO) property investments by city. Use whenever someone wants to evaluate buying an Airbnb / short-term rental / STR in a specific market — e.g. "analyze an Airbnb in Wilmington NC for $500k", "is a STR in Gatlinburg a good investment?", "cash-on-cash and hold-period return on a vacation rental".
---

# STR Investment Analyzer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All math + the curated 100+ -market dataset live server-side. This skill only gathers inputs and
calls the tools — it does **not** compute anything locally.

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

**Always ask about the holding plan — do NOT assume a 10-year hold:**
- **"How long do you plan to hold it — and do you plan to sell at the end?"**
  - If they give a horizon (e.g. "7 years"), pass it as `hold_years` and include the sale/total-return projection.
  - If they **don't plan to sell** (buy-and-hold indefinitely), focus on the recurring economics — cash flow, cash-on-cash, cap rate, break-even occupancy — and either omit the sale-based hold return or present it only as an explicit "if you sold at year N…" illustration, clearly labeled as hypothetical.
  - Only fall back to a default horizon if the user truly has no preference — and say you're assuming it.

Other inputs have sensible defaults — only ask if the user volunteers or it's relevant:
- **annual_appreciation** — decimal/yr, default `0.03`. (Suggest a conservative `0.01`–`0.02` for most markets.)
- **managed vs self-managed** → `management_fee_percent`: `0.20` (professional) vs `0` (self-managed). This is the single biggest swing — offer the comparison.

> "What markets do you cover?" → call **`list_str_markets`** (returns the full catalog grouped by
> category with a total count + asOf range). Use it to confirm coverage or suggest cities.

## Step 2 — Run the one-shot forecast (primary path)

Call **`forecast_str_market`** with the location + price. It auto-populates nightly rate &
occupancy from the bundled dataset and returns BOTH the STR cash flow (pessimistic/realistic/
optimistic) AND the hold-period return in a single call — no manual rate entry.

```
forecast_str_market({
  location: "Wilmington, NC",
  purchase_price: 500000,
  hold_years: <ask the user — do NOT default to 10>,
  annual_appreciation: 0.02,
  management_fee_percent: 0      // self-managed; omit/0.20 for professionally managed
})
```

**WARNING: if you omit `hold_years` the server silently defaults to 10 years** (with no flag
indicating it was a default). Never omit it — always pass the user-stated horizon. If the user
truly has none, pass a value AND explicitly tell them you assumed N years.

`forecast_str_market` always computes a hold return for `hold_years`. If the user isn't selling,
still pass their stated horizon (or a labeled assumption) but present the result per Step 4 — lead
with the recurring cash-flow economics, not the sale.

Optional overrides (only when the user specifies): `nightly_rate`, `occupancy_rate` (override the
bundled values), `down_payment_percent` (0.20), `closing_costs_percent` (0.03), `mortgage_rate`
(0.07), `mortgage_term_years` (30), `furniture_cost` (15000), `cleaning_fee`, `monthly_property_tax`,
`monthly_insurance`, `monthly_hoa`, `monthly_utilities`, `platform_fee_percent` (0.17),
`annual_maintenance_percent` (0.01), `index_return` (0.07 — the broad-index real return used for the
always-on opportunity-cost comparison; raise/lower it only if the user has a specific expected return).

### If the market isn't covered (`needs_market_data: true`)
`forecast_str_market` returns a soft `needs_market_data` envelope with `suggestions` (nearest
covered markets) and a `coverage_note`. **Do not proceed silently.** First tell the user their
market isn't in the dataset and show the suggestions, then ask whether to:
(a) use a suggested nearby covered market, or
(b) **web-research** the current Airbnb/VRBO average daily rate + occupancy for their exact market
(public sources: AirDNA, AirROI, AirBtics, Rabbu) and re-call `forecast_str_market` with explicit
`nightly_rate` + `occupancy_rate`.
Never invent numbers or pick a nearby market without confirming. Always label researched numbers as
estimates with their source + date — never as live quotes — and let the user override.

## Step 3 — Sensitivity / side-by-side (optional, finer control)

- **Managed vs self-managed:** run `forecast_str_market` twice, varying only `management_fee_percent`
  (`0.20` vs `0`), and compare realistic cash flow + total return. (Self-managing is usually the
  difference between a loss and a gain.)
- **Appreciation sensitivity:** vary `annual_appreciation` (e.g. 1% vs 2% vs 3%).
- **Reno / forced exit value:** for finer control, take the realistic `annual_cash_flow` and call
  **`analyze_property_return`** directly with `annual_net_cash_flow`, plus `sale_price_override` to
  set an explicit exit price (e.g. an after-renovation ARV) instead of the appreciation-derived sale.
- **Cash-flow only / custom STR expenses:** `analyze_str_property` also accepts `location` (auto-fill)
  or manual `nightly_rate`/`occupancy_rate` and the full STR expense set, if the user wants to tune
  management, cleaning, taxes, etc. before chaining to `analyze_property_return`.

## Step 4 — Present the result

From `forecast_str_market`:
- **data_provenance** — `origin` (bundled-dataset / researched / user-provided), `nightly_rate`,
  `occupancy_rate`, `source`, `source_url` (cite this), `as_of`, `confidence`, `market_key`,
  `match_type`. Always show where the numbers came from; label bundled/researched data as estimates,
  not live. NOTE: `data_provenance.confidence` is match-quality only. Surface
  **`disclosures.confidence`** as the headline confidence of the forecast; when the two diverge, lead
  with the lower one.
- **str.realistic** (also pessimistic/optimistic) — `annual_cash_flow`, `monthly_cash_flow`,
  `cash_on_cash_return_pct`, `cap_rate_pct`, `break_even_occupancy_pct`, `risk_level`, `risk_factors`.
- **str.investment_summary** — `total_cash_required`, `down_payment`, `closing_costs`, `furniture_cost`,
  `loan_amount`, `monthly_mortgage`.
- **hold_return** — `hold_years`, `total_return_pct`, `annualized_return_pct` (+ `annualized_method`:
  IRR or CAGR), `total_profit`, `sale_price`, `net_sale_proceeds`, `initial_cash_invested`,
  `total_rental_cash_flow`, and **`opportunity_cost`** (see below). (`equity_at_sale` is NOT returned
  here — it is only available via `analyze_property_return`.) **Only feature the sale-based return if
  the user plans to sell.** If they're holding indefinitely, either omit it or show it as an explicit
  "if you exited at year N…" illustration, clearly labeled as hypothetical.

**ALWAYS present the opportunity-cost comparison.** Both `forecast_str_market` (inside `hold_return`)
and `analyze_property_return` return an `opportunity_cost` block. Frame it exactly as: "Instead of
buying the STR for N years, you could park the SAME cash in a ~7% index fund for the SAME N years —
which wins?" Read and present:
- `index_terminal` (the index-fund ending wealth) vs `str_terminal` (the STR ending wealth: net sale
  proceeds + each year's net cash flow reinvested at the index rate — apples-to-apples),
- `winner` (`str` / `index` / `tie`) and `delta` (str_terminal − index_terminal),
- `str_multiple` and `index_multiple` (ending wealth ÷ initial cash),
- `breakeven_index_return` if non-null (the index return at which the two tie).
Then ALWAYS carry the honest `note`: the comparison is NOT risk-adjusted — the STR's edge comes from
leverage + sweat equity (self-management) and carries concentration, liquidity, and regulatory risk a
diversified passive index fund does not; appreciation is an assumption; figures are pre-tax. Use the
configurable `index_return` if the user has a specific expected return.

Always lead with realistic **cash-on-cash**, **cap rate**, and **break-even occupancy** (vs the
market's actual occupancy). Add the **annualized hold/total return** as the headline ONLY when a sale
is planned.

**If realistic cash flow is negative or `risk_level` is `high`, say so plainly up front — do not bury
it.** Show the management (self- vs professionally-managed) and appreciation sensitivity to reveal
whether any reasonable scenario turns it positive. Call out honest caveats: thin occupancy margins in
seasonal markets, that appreciation is an assumption, and that self-managing is real work (not "free").

## Recommended call sequence (typical session)

1. (optional) `list_str_markets` → confirm the city is covered / suggest one.
2. `forecast_str_market` { location, purchase_price, hold_years, annual_appreciation,
   management_fee_percent } → one-shot cash flow + hold return.
3. (if asked) `forecast_str_market` again with `management_fee_percent: 0` → self-managed compare.
4. (advanced) `analyze_property_return` with `sale_price_override` for a reno/forced-exit scenario.

## Notes
- All decimals are fractions, not percents: 20% → `0.20`, 65% → `0.65`.
- All figures are real (inflation-adjusted) dollars.
- The bundled dataset (AirROI/AirDNA/Rabbu public reports) is labeled with `as_of`; never present it
  as live/current quotes. Web-researched numbers are estimates too — always cite source + date.
- The opportunity-cost (STR vs index fund) comparison is NOT risk-adjusted: it ignores the STR's
  concentration, liquidity, and regulatory risk and treats appreciation as a given. Present it as a
  matched-horizon wealth comparison, not a risk-equivalent one.
- Not financial advice. Planning estimates only.
