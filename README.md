# North Carolina General Statutes

North Carolina General Statutes by citation, plus full-text topic search. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1686+ live data sources.

See `docs/state-law-probe.md` for how this source was verified — every state pack
confirms a MISS is distinguishable from a HIT before shipping, because several
state sites answer HTTP 200 for statutes that do not exist.

## Coverage (fleet #2505)

The per-section live path (`BySection/Chapter_N/GS_N-M.html`) is throttled by
ncleg.gov — measured 82 of 85 calls timed out on 2026-09-29. `nc_general_statute`
now tries a pre-fetched text first: `scripts/ingest-nc-statutes.mjs` fetches ncleg.gov's
own per-CHAPTER bulk documents (`ByChapter/Chapter_N.html` — 396 requests,
counting lettered subchapters, not ~34,000 per-section requests) and splits
each on its `§ N-M.  Heading.` markers, writing one JSON bundle per chapter
into the shared `pipeworx-datasets` R2 bucket at `statutes/nc/<chapter>.json`.

If the chapter has been crawled and the section is simply not in it, that is
authoritative (repealed/renumbered sections are normal) and the tool says so
without a live round trip. If the chapter has not been crawled yet, it falls
back to the original per-section live fetch.

`data_as_of` on every response says whether it came from the pre-fetched text (the
crawl's `ingested_at`) or a live fetch (now).

## Topic search

`nc_statute_search` answers questions no citation lookup can — "election
conduct at polling places", "landlord retaliatory eviction" — via FTS5 in the
shared SEARCH_SHARD Durable Object (`workers/gateway/src/search-shard.ts`,
shard name `nc-statutes`; no new Cloudflare binding). Search hits carry a
citation and heading; call `nc_general_statute` with the section from a result
to get the full text.

## Known limitation: "Reserved" sub-section headings

The chapter splitter anchors on `§ <section>.  <Heading>` — a literal period
before the heading. A minority of sections use a colon instead, specifically
dotted sub-numbers marking a placeholder: `§ 1-87.2:  Reserved for future
codification purposes.` The splitter's period-anchor backtracks onto the dot
inside "1-87.2" and folds it under the base section (`1-87`), so eleven
placeholder sub-sections can attach to one real section's bundle entry as
separate array items sharing its number. Measured across the full 2026-09-30
crawl: 460 (chapter, section) pairs affected, 1,875 extra entries, essentially
all "Reserved for future codification purposes" (verified — no substantive
section heading appeared in a sample of the affected pairs). A real citation
lookup for one of these numbers still resolves (the FIRST matching entry,
which is the genuine section); only the index used for search-hit resolution
carries the redundant placeholder rows, and the search index itself remains
correct rank-wise since a placeholder has no meaningful full-text content to
rank on. Not fixed here because the affected content is exclusively "reserved,
no text" filler; fixing the splitter to disambiguate `.` inside a dotted
sub-number from `.` as a heading terminator is a bounded follow-up if a real
citation ever collides this way.

## Refresh

`node scripts/ingest-nc-statutes.mjs` — re-run periodically (NC's legislative
session runs annually; there is no daily-freshness need here). No schedule is
wired up: dispatch it by hand, or via `gh workflow run` if/when a workflow is
added — GitHub's own `schedule:` trigger has been unreliable since ~2026-09-09
(see CLAUDE.md), so a recurring refresh should follow the monitor-dispatch
pattern rather than a bare cron trigger.

## Data source

Official state legislature site (ncleg.gov). North Carolina statutes are
public record; no reuse restriction was found on the bulk document paths used
here.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "north-carolina-code": {
      "url": "https://gateway.pipeworx.io/north-carolina-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/north-carolina-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1686+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/nc_general_statute \
  -H 'Content-Type: application/json' \
  -d '{"section":"14-17"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nc_general_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "north-carolina-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-north-carolina-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-north-carolina-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about North Carolina Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
