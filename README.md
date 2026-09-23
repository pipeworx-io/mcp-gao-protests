# @pipeworx/gao-protests

GAO bid-protest decisions — government-contract award disputes ruled on by the
US Government Accountability Office ("did anyone protest this award", "how
often does this agency lose protests"). Pairs with `usaspending` and
`dod-contract-announcements`, which cover the award side but not disputes.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1663+ live data sources.

## Tools

- `gao_protests_recent(days?, outcome?, limit?)` — recently decided protests, optionally filtered by outcome (denied|dismissed|sustained).
- `gao_protest(b_number)` — a single decision by B-number, with the decision PDF link.
- `gao_protests_search(protester, limit?)` — search accumulated decisions by protester/company name.
- `gao_protests_coverage()` — total decisions held, earliest/most-recent decided_on, last crawl time.

## Auth

Keyless — no operator key needed. Backed by `gao_bid_protests`, refreshed
daily by `workers/data-pipeline` (dataset `gao-protests`).

## Data sources

- <https://www.gao.gov/legal/bid-protests/search> — the bid-protests landing
  page. It embeds a Drupal Views "recent decisions" block directly in the
  page HTML (no JS challenge), showing the ~5 most-recent decisions.
- <https://www.gao.gov/products/B-XXXXXX> — each decision's own page, fetched
  once per new B-number for its exact decided date and PDF link.

**Coverage is a rolling accumulation, not GAO's full protest archive.** The
filtered search endpoint (`gao.gov/search?f[0]=ctype_search:Bid Protest
Decision`) sits behind an Akamai Bot Manager proof-of-work JS interstitial
that cannot be solved from a server — full Chrome headers over HTTP/2 don't
help, this is a JS-execution challenge, not a header fingerprint. The landing
page's own "More" link points at that same blocked endpoint, and there is no
other reachable pagination. So this pack can only ever see the decisions GAO
has published *since it started crawling* — `gao_protests_recent`/`_search`
answer against what has accumulated in `gao_bid_protests` so far, not against
GAO's complete decision history. `gao_protests_coverage` reports exactly how
far back that goes.

**Both hosts need the full Chrome header set** (`User-Agent`, `Accept`,
`Accept-Language`, `sec-ch-ua*`, `Sec-Fetch-*`, `Upgrade-Insecure-Requests`
over HTTP/2) — a plain `User-Agent` alone gets a 403. This is a
header-fingerprint block, not a Cloudflare-egress block; no proxy is needed.

**The protested AGENCY is not available.** It appears only inside each
decision's PDF text, never in the page HTML, and this pipeline has no PDF
text extraction. Rather than guess an agency from context (or from the
protester's history), the field is omitted entirely — presenting a guessed
agency as fact would be worse than not having it.

**A decision can carry several B-numbers** (e.g. `B-424484,B-424484.2` for an
original protest plus a reconsideration). The table's primary key is the
first number in that string; `gao_protest` also matches a non-primary number
(like the `.2` suffix) by searching the raw string.

**Outcome is prose, not an enum** ("We deny the protest.", "We dismiss the
request for reconsideration.", "We sustain the protest."). It's normalized to
`denied | dismissed | sustained | other` for filtering, and the raw sentence
is always kept alongside it.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "gao-protests": {
      "url": "https://gateway.pipeworx.io/gao-protests/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/gao-protests/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1663+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/gao_protests_recent \
  -H 'Content-Type: application/json' \
  -d '{"days":30}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/gao_protests_recent`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "gao-protests": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-gao-protests"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-gao-protests
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Gao Protests data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
