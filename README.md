# @pipeworx/sec-fundamentals

Point-in-time US public company financials with no look-ahead: what a company's statements said on a given date, using only values it had filed by then. Also every restatement of a line, and cross-company screens of one concept and one period. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2036+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `sec_fundamentals_as_of(company, as_of?, concepts?, period?, periods?)` — one company's standard statement lines as known on `as_of`. A restatement filed after `as_of` cannot change the answer. Returns value, period start and end, calendar period label, filing date, form, accession, and whether the period had already been restated by `as_of`.
- `sec_fundamentals_restatements(company, concept, period_end?, as_of?, include_unrevised?, limit?)` — every reporting period of one line whose value changed between filings, with each version, filing date and the change in absolute and percent terms.
- `sec_fundamentals_frame(concept, period, as_of?, sort?, min_value?, max_value?, limit?)` — every filer's value of one concept in one calendar period (`CY2023`, `CY2023Q1`, `CY2023Q4I`), ranked, as known on `as_of`. One row per company.
- `sec_fundamentals_concepts(statement?, search?)` — the 76 standard concepts (income statement, balance sheet, cash flow, per-share and cover-page lines), their units and the ordered XBRL tag chain each is read from.

`edgar_company_concept` is the live single-company lookup (with its own `as_of`, restated flags and derived Q4). This pack is the bulk, screenable store: a whole statement set as of a past date in one call, and screens across every filer.

## Auth

Keyless.

## Data sources

- US Securities and Exchange Commission, XBRL financial data. US federal government data, public domain. Covers filings from 2009 onward (the start of XBRL) by operating companies that report any of the standard concepts, about 11,000 filers.

Every response carries `data_as_of` (the newest filing date in the data) and `source`.

## How the data is shaped

- **One record per filer** holds every distinct value of each standard concept, with the filing that first reported it: filed date, form, accession, period start and end, and the XBRL tag it came from. A later filing that repeats the same number is not stored (it adds nothing to an as-of read); a later filing with a different number is stored, and that is the restatement.
- **Tag chains.** Each concept maps to an ordered list of XBRL tags (revenue: `RevenueFromContractWithCustomerExcludingAssessedTax`, then `Revenues`, then `SalesRevenueNet`, and so on). For each reporting window the first tag in the chain that the filer reports wins, so a company that changed tags still gets one continuous series. The map is in `src/concepts.ts` and is returned by `sec_fundamentals_concepts`.
- **Frames.** For cross-company screens, every value is also placed on a calendar period in the SEC frames convention: a full year (about 365 days) is `CYyyyy`, a discrete quarter (about 91 days) is `CYyyyyQn`, a balance at a date is `CYyyyyQnI`. Fiscal periods land in the nearest calendar period. Six- and nine-month year-to-date cash-flow windows are in company records (`period: "all"`) but not in frames.
- **No look-ahead.** `as_of` filters on the filing date, never the period date. A period first filed after `as_of` is absent; a figure restated after `as_of` shows its earlier value.
- **Explicit empty answers.** An `as_of` before the company's first filing returns `found:false`, `empty_reason: "as_of_before_first_filing"` and the earliest filing date. A concept the company does not report returns `empty_reason: "concept_not_reported"`.

### Things to know

- `fiscal year` and `fiscal period` in the raw SEC data describe the FILING, not the period a value covers (a 10-Q for fiscal Q3 also reports a prior-year balance). Rows here carry the period's own start and end and a calendar `period` label instead.
- Discrete Q4 is not a filed figure and is not derived here. Use `edgar_company_concept` for a derived Q4 on a live lookup.
- The first version of an older period is the earliest XBRL-era filing that carried it, not necessarily the original pre-XBRL report.
- Stock splits appear as restatements of per-share and share-count lines, because later filings recast them.

## Budget

- One-time load: about 11,000 company objects plus about 8,000 frame objects plus 2 small ones, about 19,000 writes at $4.50 per million = about $0.09, plus list calls; target $0.11.
- Storage about 0.5 GB at $0.015 per GB-month = under $0.01 per month.
- Refresh: at most daily, rewriting only objects whose bytes changed (gzip output is deterministic and the new md5 is compared with the stored one). A day's filings touch a few hundred company objects and the frame periods they land in: about 1,000 writes a day = about $0.14 per month, plus the listing. Under $1 per month in total.
- Zero new bindings, no database, no Durable Object. Reads are one to three object reads per call.

`scripts/load-sec-fundamentals.mjs` carries the same budget as a `# BUDGET:` header.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sec-fundamentals": {
      "url": "https://gateway.pipeworx.io/sec-fundamentals/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sec-fundamentals/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2036+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/sec_fundamentals_as_of \
  -H 'Content-Type: application/json' \
  -d '{"company":"AAPL","as_of":"2010-02-01","concepts":["revenue","net_income","total_assets"],"period":"annual","periods":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/sec_fundamentals_as_of`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-sec-fundamentals` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/sec-fundamentals/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "sec-fundamentals": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sec-fundamentals"]
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
ask_pipeworx({ question: "your question about Sec Fundamentals data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
