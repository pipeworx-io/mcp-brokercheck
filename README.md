# @pipeworx/brokercheck

Look up whether a stockbroker, investment-adviser rep, or brokerage firm has a
clean regulatory record, via FINRA BrokerCheck — the public professional-conduct
registry FINRA itself publishes for investor due diligence.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `brokercheck_search_individual(query, limit?)` — search individuals by name;
  returns CRD, scope, `has_disclosures` flag, current employer(s).
- `brokercheck_individual_detail(crd)` — full profile by CRD, including the
  actual disclosure EVENTS (type, date, resolution, detail) when present —
  not just the flag.
- `brokercheck_search_firm(query, limit?)` — search brokerage firms by name;
  returns CRD, SEC number, scope, `has_disclosures` flag, branch count.

**Per-name lookup only, by design.** No bulk-enumeration tool is offered —
see the Terms-of-Use finding below.

## Auth

Keyless. `api.brokercheck.finra.org` is the same public API BrokerCheck's own
search box calls; no registration or key required.

## Terms of Use finding (read before touching this pack)

Quoted from `https://brokercheck.finra.org/terms` (BrokerCheck Terms of Use)
and its plain-language summary at
`https://www.finra.org/investors/learn-to-invest/choosing-investment-professional/about-brokercheck/permitted-uses`,
both confirmed live 2026-09-13:

> Users agree to comply with BrokerCheck's Terms of Use, which permit use of
> BrokerCheck data for **investor protection, academic, compliance or
> regulatory purposes**. The Terms of Use do not permit use of BrokerCheck for
> other purposes (with limited exceptions)... You can use BrokerCheck for
> personal and professional use to **find an investment professional; review
> the background of your current investment professional**; assist in
> judicial or arbitral proceedings; comply with securities or financial
> services laws, rules and regulations; or for similar uses consistent with
> promotion of just and equitable principles of trade and protection of
> investors and public interest.

> If your use is for investor protection, academic, compliance or regulatory
> purposes, you may copy and compile BrokerCheck data, including by using data
> mining or similar tools, **provided that such tools do not interfere with
> the proper working of BrokerCheck**.

**Redistribution requirements** (apply when BrokerCheck data is passed to a
third party): identify BrokerCheck as the source; provide links to BrokerCheck
and its Terms of Use; notify the recipient their use is subject to the Terms
of Use; maintain a correction process; disclose when the data was compiled;
keep the data current. **Key prohibitions**: do not alter/modify the factual
content of BrokerCheck data; do not use BrokerCheck data for unsolicited
marketing of goods or services.

**How this pack complies:**

- **Purpose fits the permitted-use categories.** Every tool answers
  "find/review the background of an investment professional or firm" —
  exactly the personal/professional due-diligence use the Terms of Use name —
  and every tool description says so.
- **No caching, no storage, no bulk redistribution.** Every call is a live,
  per-request pass-through to `api.brokercheck.finra.org` — nothing is copied,
  compiled, or republished as a static or bulk dataset. This is closer to a
  real-time embedded search than the "copy and compile" / data-mining use the
  redistribution clause is written for, but every response still carries the
  attribution block below out of caution.
- **Attribution block on every response**: `source: "FINRA BrokerCheck"`,
  `source_profile_url` (a live link to the individual's/firm's BrokerCheck
  profile page), `terms_of_use` (link to the Terms of Use), and `compiled_at`
  (an ISO timestamp — this data was compiled *now*, at call time, since it is
  a live pass-through and never stale).
  - `terms_of_use_notice`: callers relaying this data to an end user should
    notify them their use is subject to the BrokerCheck Terms of Use above.
- **No alteration of factual content.** Field values are passed through
  as-returned (renamed/restructured for consistency with our other packs,
  never reworded or reinterpreted).
- **No marketing use.** Nothing in this pack's design or description
  encourages using BrokerCheck data for unsolicited marketing.
- **Per-name lookup only — no bulk enumeration tool.** There is no
  "list all brokers" or "enumerate CRDs" tool. `limit` on the two search tools
  caps a single name-query's results at 50, mirroring what BrokerCheck's own
  UI returns; it does not enable crawling the registry.
- **Rate/load**: no query-string flags weaken FINRA's own defaults; the pack
  makes exactly one upstream request per tool call it services, so caller
  request volume already tracks Pipeworx's own per-account rate limits rather
  than adding an independent crawl surface.

**Conclusion: ships.** The serving shape (live per-name lookup, sourced,
timestamped, unaltered, non-bulk) sits inside the permitted-use language
above. This is not a categorical clearance for every possible use of this
pack — a caller who tried to enumerate the whole registry through repeated
per-name calls would be outside these terms regardless of what this README
says, which is exactly why no bulk tool exists.

## Data sources

- `https://api.brokercheck.finra.org/search/individual` — individual search
  (`ind_*` fields; disclosure flag is `ind_bc_disclosure_fl`).
- `https://api.brokercheck.finra.org/search/individual/{crd}` — individual
  detail; response wraps everything in a JSON **string** under `content`
  (parse it) containing `basicInformation`, `currentEmployments`,
  `previousEmployments`, and `disclosures` (each disclosure has `eventDate`,
  `disclosureType`, `disclosureResolution`, `disclosureDetail`).
- `https://api.brokercheck.finra.org/search/firm` — firm search (`firm_*`
  fields; disclosure flag is `firm_disclosure_fl`).
- Public profile pages for citation: `https://brokercheck.finra.org/individual/summary/<crd>`,
  `https://brokercheck.finra.org/firm/summary/<crd>`.

Gotcha: this is a *separate* API and a *separate* licence from the existing
`finra` pack (short-sale/equity market data, its own strict FINRA licence
terms) — do not fold the two together, and do not assume finra's licence
terms apply here or vice versa.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "brokercheck": {
      "url": "https://gateway.pipeworx.io/brokercheck/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/brokercheck/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/brokercheck_search_individual \
  -H 'Content-Type: application/json' \
  -d '{"query":"John Smith"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/brokercheck_search_individual`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "brokercheck": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-brokercheck"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-brokercheck
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Brokercheck data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
