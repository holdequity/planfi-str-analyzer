# STR Investment Analyzer (Claude Agent Skill)

Analyze short-term-rental (Airbnb / VRBO) property investments by city, right inside Claude.
Ask something like _"analyze an Airbnb in Wilmington NC for $500k"_ and the skill walks you
through a couple of inputs and returns a full forecast: cash flow, cash-on-cash, cap rate,
break-even occupancy, and a hold-period return for the horizon you choose (it asks how long
you'll hold — it does not assume 10 years).

It's a **thin orchestration layer** over the public **planfi MCP** — all the math and the
curated market dataset live server-side. The skill itself bundles no engine; it just gathers
inputs and calls the tools.

## What you need

The skill calls these planfi MCP tools, served from `https://ai.planfi.app/mcp/free` (public, no auth):

- **`forecast_str_market`** — the primary one-shot tool. Give it a covered city + price and it
  auto-populates nightly rate & occupancy from the bundled dataset (100+ markets) and returns the
  STR cash flow (pessimistic/realistic/optimistic) **and** the hold-period return in a single call.
- **`list_str_markets`** — the coverage catalog (all bundled markets + count + as-of range).
- **`analyze_str_property`** — finer STR control (accepts `location` to auto-fill, or manual
  `nightly_rate`/`occupancy_rate`, plus the full STR expense set incl. `management_fee_percent`).
- **`analyze_property_return`** — hold-period return with finer control (incl. `sale_price_override`
  for a forced/after-renovation exit price).
- **`analyze_str_tax_loophole`** — the §469 short-term-rental "loophole": tests whether average guest
  stay ≤ 7 days (non-rental classification) plus material participation makes your year-1
  cost-seg/bonus-depreciation loss **non-passive** and deductible against W-2 income, quantifies the
  ordinary tax saved, and projects the §1250/§1245 depreciation-recapture liability at sale. If
  material participation fails it reports the suspended loss and routes to `analyze_passive_losses`.

If the city is **covered** by the bundled dataset, rate + occupancy auto-populate server-side — no
manual entry. If it's **not** covered, `forecast_str_market` returns `needs_market_data` with nearby
suggestions; the skill then asks whether to use a suggested market or web-research ADR + occupancy
for your exact market (labeled as estimates with source + date).

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp/free` (no auth). The skill also reminds you to do this if the tools
are missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-str-analyzer
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/str-investment-analyzer ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/str-investment-analyzer .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-str-analyzer
/plugin install str-investment-analyzer@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r str-investment-analyzer.zip str-investment-analyzer`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.
   (Your account must have Skills enabled.)

## Example prompts

- "analyze an Airbnb in Wilmington NC for $500k"
- "is a short-term rental in Gatlinburg a good investment if I pay $650k and self-manage?"
- "compare cash-on-cash for a $400k Airbnb in Scottsdale, managed vs self-managed, then show the
  return if I sold after 7 years at 2% appreciation"
- "I make $300k W-2, my Airbnb's average guest stay is 4 nights, I logged 180 hours and a $120k
  cost-seg loss — can I write that off against my salary, and what's the recapture when I sell?"

## How it works (call flow)

1. (optional) `list_str_markets` → confirm the city is covered / suggest one.
2. `forecast_str_market { location, purchase_price }` → one-shot: bundled rate/occupancy, cash-flow
   scenarios, break-even, a top-level risk flag, the managed-vs-self comparison (`self_managed_comparison`,
   no second call), data provenance, and the hold-period return. Pass `hold_years` only if you have a
   sale horizon — omit it for an indefinite buy-and-hold and the server labels the sale return as an
   illustrative hypothetical (it never silently assumes 10 years).
3. (advanced) `analyze_property_return` with `sale_price_override` for a reno/forced-exit scenario.

If the market isn't in the bundled dataset, the skill surfaces the `needs_market_data` suggestions
and asks before web-researching ADR + occupancy and re-calling with explicit `nightly_rate` +
`occupancy_rate`.

See `SKILL.md` for the full instructions, exact tool params, and output format.

## Notes & honest caveats

- Bundled market numbers (nightly rate, occupancy) are **estimates from public reports
  (AirROI/AirDNA/Rabbu) with a source + as-of date**, not live quotes — the skill always labels
  them. Web-researched fallback numbers are estimates too, with source + date.
- STR cash flow is sensitive to occupancy; always check the break-even occupancy and the
  pessimistic scenario, not just the realistic one. If a deal is negative or high-risk, the skill
  says so up front.
- Not financial advice. Planning estimates only.

## License

MIT — see [LICENSE](./LICENSE).
