# @pipeworx/earthscope-fdsn

Seismic station metadata, federated waveform holdings and regional earthquake
catalogues, over the international FDSN web-service standard.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

This is the **instrument** side of seismology, not the global earthquake
catalogue — `usgs-earthquakes` and `emsc` already cover "what earthquakes
happened". `fdsn_events` here adds the REGIONAL agency catalogues, which list
the local magnitude-1-to-3 seismicity the global catalogues never publish.

## Tools

- `fdsn_stations(network?, station?, channel?, level?, lat/lon/radius or bbox, …)` —
  where seismometers are, what they record, who operates them, and when.
- `fdsn_events(datacenter?, starttime?, min_magnitude?, …)` — regional agency
  earthquake catalogues: `gfz` (global rapid solutions), `ingv` (Italy),
  `resif` (France), `koeri` (Turkey).
- `fdsn_availability(network, station?, channel?, starttime?, endtime?)` — which FDSN
  data centres hold waveform data for a channel and time window, with each
  centre's dataselect URL.

## Auth

Keyless.

## Data sources

- <https://service.earthscope.org/fdsnws/station/1/query> — station metadata.
- <https://service.earthscope.org/irisws/fedcatalog/1/query> — federated holdings.
- <https://geofon.gfz.de/fdsnws/event/1/query>, <https://webservices.ingv.it/fdsnws/event/1/query>,
  <https://api.franceseisme.fr/fdsnws/event/1/query>,
  <http://www.koeri.boun.edu.tr/services/fdsnws/event/1/query> — regional catalogues.

### Things the next person would otherwise rediscover (all verified 2026-09-17)

- **`service.iris.edu` 301-redirects to `service.earthscope.org`.** Both work;
  the old host costs a redirect on every call, so the new one is used directly.
- **EarthScope serves only `station` and `dataselect` under `/fdsnws/`.** Their
  `event` and `availability` services answer HTTP 404. Availability is behind an
  EarthScope login — it redirects to a trailing-slash path served by `uvicorn`
  with `vary: Cookie`. That is why `fdsn_availability` answers from
  `irisws/fedcatalog` instead: keyless, and it reports the same holdings
  question across every federated data centre rather than one.
- **`nodata=404` makes an empty result indistinguishable from a retired
  endpoint** — both are a bare 404 with an empty body, which is exactly how the
  retired event service was nearly mistaken for "no earthquakes". These tools
  send `nodata=204`, so an empty result is a 204 and a 404 genuinely means the
  route is gone.
- **Field names differ between `level=station` and `level=channel`**, and the
  depth column is spelled `Depth/km` at one agency and `Depth/Km` at another, so
  rows are read by header name with aliases rather than by position.
- **`fedcatalog` answers in its own `format=request` dialect**, not FDSN text:
  blank-line-separated blocks, each opening with `KEY=value` service lines and
  then bare `NET STA LOC CHA START END` rows.
- **FDSN `maxradius` is in DEGREES of arc**, not km. The tools take km and
  convert (÷ 111.195).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "earthscope-fdsn": {
      "url": "https://gateway.pipeworx.io/earthscope-fdsn/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/earthscope-fdsn/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/fdsn_stations \
  -H 'Content-Type: application/json' \
  -d '{"network":"IU","station":"ANMO","level":"channel","channel":"BH*"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/fdsn_stations`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "earthscope-fdsn": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-earthscope-fdsn"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-earthscope-fdsn
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Earthscope Fdsn data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
