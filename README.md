# @pipeworx/usda-market-news

USDA Agricultural Marketing Service, Market News: livestock price reports (direct slaughter cattle, boxed beef and cow-beef cutouts, hogs and pork, lamb) and, with a registered key, the My Market News reports for fruit and vegetables (shipping-point, terminal-market and import-crossing prices), grain, hay, cotton and ornamentals. Dated rows with the report's own basis (class, selling basis, grade, weight range) and low / high / weighted-average prices kept as separate fields.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ams_report_search(query?, species?, limit?)` — finds livestock price reports by title words and species; returns report id, report code, the sections each report has, and when AMS last published it. A search with no match says what is not covered instead of returning an empty list.
- `ams_report(report_id, section?, date?, start_date?, end_date?, limit?)` — reads rows from one report section. Every response states the report family, report title, market type and office. With no date it returns the most recent report date. A date or section AMS has no rows for returns `status: "no_rows_for_request"` with the report's last published date, never an empty array or a zero.

- `ams_market_report_search(query?, market_type?, limit?)` -- finds My Market News reports outside livestock (produce shipping point and terminal markets, Mexico crossings, grain bids, hay, cotton) by words in the title, market type or office. A search with no match returns the market types that exist, never a bare empty list. Commodities and crossing districts are fields inside a report's rows, not in its title: find the office report here, then filter its rows in `ams_market_report`.
- `ams_market_report(report_id, commodity?, district?, section?, date?, start_date?, end_date?, limit?)` -- reads dated rows from one such report, optionally keeping one commodity (for example lime) and one district or market location (for example mexico crossings through texas). Reports are daily; a weekly view is a start_date/end_date range of up to 31 days. Low / high / mostly / mean prices stay separate fields with the package or unit AMS gives. A date or commodity with no rows returns a status that says so, with the commodities the report does carry.

## Auth

The two livestock tools are keyless. The two My Market News tools use the Pipeworx platform key for the AMS My Market News registered service; callers can pass their own free key as `_apiKey`.

## What is NOT covered

Poultry and egg reports, and anything AMS does not publish through Market News. Livestock reports are `ams_report*`; everything else AMS publishes in Market News is `ams_market_report*`.

## Reading prices correctly

- **Families are not interchangeable.** A live or dressed direct-slaughter cattle price (class, selling basis `LIVE FOB` / `DRESSED DELIVERED`, grade), a boxed-beef cutout (Choice/Select, 600-900 lb carcass-cut value) and a pork cutout (FOB plant vs FOB Omaha) are different quantities even for the same animal. Read the basis fields on each row before comparing.
- **Ranges stay ranges.** `price_range_low`, `price_range_high` and `weighted_avg_price` are separate fields exactly as AMS publishes them. Nothing here averages or rolls up.
- **Units.** The data carries no unit field. AMS quotes LMR prices in dollars per hundredweight and weights in pounds; each response repeats this as `unit_note`.
- **Numbers are strings** as AMS publishes them (`"63,231"`, `"369.47"`). Null fields are dropped from rows.
- **Dates.** ISO in and out of the tool arguments; `report_date` on rows is as AMS writes it (MM/DD/YYYY).

## Data sources

- USDA Agricultural Marketing Service, My Market News (specialty crops, grain, hay, cotton, ornamentals). US federal data.
- USDA Agricultural Marketing Service, Livestock Mandatory Price Reporting (Market News). US federal data.

Behaviour worth knowing: a request for a date AMS has not published answers with a plain "No Results Found." string rather than an empty list, which is how `no_rows_for_request` is detected. Weekly reports publish on one weekday only, so a daily date will often come back empty for them. The same report id has different row shapes in different sections (head counts in `Summary`, prices in `Detail` for most cattle reports).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "usda-market-news": {
      "url": "https://gateway.pipeworx.io/usda-market-news/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/usda-market-news/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ams_report_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"boxed beef cutout","species":"beef"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ams_report_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-usda-market-news` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/usda-market-news/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "usda-market-news": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-usda-market-news"]
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
ask_pipeworx({ question: "your question about Usda Market News data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
