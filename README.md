# @pipeworx/medicaid-spending

What each state spends on Medicaid, by service category and federal fiscal year, split into federal and state share (CMS CMS-64 net expenditures, fiscal years 2014-2024), the federal matching rate (FMAP and enhanced FMAP) by state and fiscal year 2014-2027, and the quarterly expansion-group and temporary increased-FMAP expenditure reports.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2015+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `medicaid_spending_by_state(state, fiscal_year?, program?, category?, include_chip?, limit?)` — one state's (or the nation's) headline total net expenditures and its service-category rows (managed care, prescribed drugs, nursing facility, home and community-based services, and so on), each with total computable, federal share, state share and the federal share percentage.
- `medicaid_spending_trend(state, category?, program?)` — the same figures for every fiscal year 2014-2024, with year-over-year growth. Defaults to the headline total; pass a category to trend one service line.
- `medicaid_spending_ranking(fiscal_year?, sort_by?, category?, program?, limit?)` — all 56 jurisdictions ranked by total, federal share, state share or federal share percentage, beside the national row.
- `medicaid_fmap(state?, fiscal_year?)` — the Federal Medical Assistance Percentage and enhanced FMAP, for one state across years, or every jurisdiction in one year.
- `medicaid_expansion_spending(state?, quarter_end?, limit?)` — quarterly spending on the ACA expansion ("new adult") group: total computable and federal share, newly eligible versus not, beside all-medical-assistance.
- `medicaid_increased_fmap_spending(program?, state?, quarter_end?, limit?)` — quarterly expenditure under the temporary increased match periods (the 2020 pandemic increase, and the phase-down under the 2023 appropriations act).

A state, fiscal year, quarter or category that matches nothing returns the valid values and near matches, never a bare empty result.

## Auth

Keyless.

## Data sources

- CMS, Medicaid Financial Management Report (CMS-64 net expenditures): one workbook per federal fiscal year, per state and national totals, for the Medical Assistance Program (benefit spending) and Administration. Fiscal years 2014 to 2024 are served. States report these figures to CMS on the CMS-64; the report is net of collections and rebates.
- HHS, annual Federal Medical Assistance Percentages notice in the Federal Register: one notice per fiscal year (2014 to 2027 served).
- CMS, Medicaid CMS-64 New Adult Group Expenditures, and CMS, Medicaid CMS-64 increased-FMAP expenditure reports.

The expenditure workbooks and the yearly FMAP notices are bulk publications with no per-state query, so those two tables are read once by `scripts/bake-index.mjs` and stored (about 2.5 MB: 64,756 non-zero rows and 784 FMAP rows). Every response carries `data_as_of`. The two quarterly datasets are read live per request. Refresh is yearly (budget in `scripts/baked-index-freshness.json`): add the new fiscal year to `FMR_YEARS` in the script and re-run it.

Checks the bake performs: for every fiscal year the state and territory sheets sum exactly to the national-totals sheet (0.000 percent difference), the national total is in the hundreds of billions, and every FMAP is between 50 and 83 percent with the enhanced rate at or above it. The script refuses to write if any of them fails.

## Notes

- Figures are whole dollars, net of offsets. Service rows include negative offsets (drug rebates), so they do not all add up positively; the "Total Net Expenditures" row is the headline. Subtotal rows ("Total ...", "Balance") are listed only when a category filter names them.
- "C-" rows are CHIP-allotment-funded Medicaid expansion expenditures and "T-" rows are the CHIP-allotment portion of qualifying-state expenditures; both sit outside the headline and are hidden unless `include_chip` is set. The separate CHIP (title XXI) publication is not included.
- The newest fiscal year can still be revised as states file adjustments. The FY2024 Texas total, for example, reads 17 percent below FY2023; check `data_as_of` before reading a one-year change as a trend.
- FMAP rows are the standard rates only. The temporary pandemic increase, the 90 percent expansion-adult rate and disaster-recovery adjustments are not in the table. Federal Register correction notices are skipped; the original notice's table is used.
- Annual Disproportionate Share Hospital allotments are **not served**: CMS publishes them in Federal Register notices as page images, with no structured source. State DSH payments inside the expenditure series appear as the "Inpatient Hospital - DSH" and "Mental Health Facility - DSH" categories.
- Territories (American Samoa, Guam, the Northern Mariana Islands, Puerto Rico, the Virgin Islands) report on the same basis; their FMAPs are statutory rates.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "medicaid-spending": {
      "url": "https://gateway.pipeworx.io/medicaid-spending/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/medicaid-spending/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 2015+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/medicaid_spending_by_state \
  -H 'Content-Type: application/json' \
  -d '{"state":"California","fiscal_year":2024,"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/medicaid_spending_by_state`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-medicaid-spending` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/medicaid-spending/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "medicaid-spending": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-medicaid-spending"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Medicaid Spending data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
