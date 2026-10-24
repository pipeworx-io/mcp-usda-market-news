# @pipeworx/usda-market-news

USDA AMS livestock price reports from Market News: direct slaughter cattle, boxed beef and cow-beef cutouts, hogs and pork, lamb. Dated rows with the report's own basis (class, selling basis, grade, weight range) and low / high / weighted-average prices kept as separate fields.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1878+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ams_report_search(query?, species?, limit?)` — finds livestock price reports by title words and species; returns report id, report code, the sections each report has, and when AMS last published it. A search with no match says what is not covered instead of returning an empty list.
- `ams_report(report_id, section?, date?, start_date?, end_date?, limit?)` — reads rows from one report section. Every response states the report family, report title, market type and office. With no date it returns the most recent report date. A date or section AMS has no rows for returns `status: "no_rows_for_request"` with the report's last published date, never an empty array or a zero.

## Auth

Keyless.

## What is NOT covered

Produce shipping-point and terminal-market prices (limes, tomatoes, onions and the rest of Specialty Crops Market News), grain, and poultry/egg reports are a different AMS service that requires a registered account key. They are not served by this pack. Asking for them here returns a `no_match` that says so. Serving them needs a key registered by a person (a Bruce step); nothing in this pack pretends to.

## Reading prices correctly

- **Families are not interchangeable.** A live or dressed direct-slaughter cattle price (class, selling basis `LIVE FOB` / `DRESSED DELIVERED`, grade), a boxed-beef cutout (Choice/Select, 600-900 lb carcass-cut value) and a pork cutout (FOB plant vs FOB Omaha) are different quantities even for the same animal. Read the basis fields on each row before comparing.
- **Ranges stay ranges.** `price_range_low`, `price_range_high` and `weighted_avg_price` are separate fields exactly as AMS publishes them. Nothing here averages or rolls up.
- **Units.** The data carries no unit field. AMS quotes LMR prices in dollars per hundredweight and weights in pounds; each response repeats this as `unit_note`.
- **Numbers are strings** as AMS publishes them (`"63,231"`, `"369.47"`). Null fields are dropped from rows.
- **Dates.** ISO in and out of the tool arguments; `report_date` on rows is as AMS writes it (MM/DD/YYYY).

## Data sources

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

Both URLs reach the same gateway and the same 1878+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ams_report_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"boxed beef cutout","species":"beef"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ams_report_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

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

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-usda-market-news
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

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
