---
layout: content.njk
contentType: blog
title: ai-catalog — One URL, Many Protocols
description: A single /.well-known/ai-catalog.json enumerates every protocol surface an agent exposes — A2A, MCP, HTTPS — under one endpoint. The wrapping is the load-bearing idea.
date: 2026-05-16
tags: [dns-aid, ai agents, ai-catalog, agent cards, multi-protocol]
hero: ./src/assets/img/posts/ai-catalog/hero.png
heroAlt: Header of the AI Catalog Unofficial Draft (30 April 2026) at agent-card.github.io/ai-catalog/, showing the spec title, draft date, latest editor's draft URL, and Creative Commons Attribution 4.0 licensing footer
heroVariant: contain
list: yes
---

Once you accept that agent cards are the right format for self-description, a second-order problem shows up almost immediately: most agents have more than one way to be called. The same agent might speak [A2A](https://a2aproject.dev/) natively, expose an [MCP](https://modelcontextprotocol.io/) server for IDE-style consumption, and answer plain HTTPS JSON for the curious. Three protocols, one logical agent. Three separate card.json files at three separate `/.well-known/` paths is awkward; one card per protocol with a different `interfaces[]` per file even more so.

The `ai-catalog` spec — currently a [GitHub-hosted draft](https://agent-card.github.io/ai-catalog/) co-developed alongside the agent-card schema — is the wrapper that fell out of that observation. One URL: `/.well-known/ai-catalog.json`. Inside it: an array of protocol-specific card entries. One catalog, many protocols. The agent advertises itself once and lets the caller pick the surface.

This is one of those formats that's easier to grasp once you see the shape, so let's look at the shape.

<br>

## What the Catalog Looks Like

A minimal catalog with three protocol surfaces:
```json

{
  "version": "1.0",
  "name": "bookings",
  "provider": { "organization": "Darknetian" },
  "catalog": [
    {
      "protocol": "a2a",
      "card": "/.well-known/agent-card/bookings.a2a.json",
      "url": "https://bookings.darknetian.com/a2a"
    },
    {
      "protocol": "mcp",
      "card": "/.well-known/agent-card/bookings.mcp.json",
      "url": "https://bookings.darknetian.com/mcp"
    },
    {
      "protocol": "https",
      "url": "https://bookings.darknetian.com/ask"
    }
  ]
}
```

A few things that matter:

- **One catalog per endpoint.** The agent host serves *one* `ai-catalog.json`. That's the entry point.
- **N entries per catalog.** Each entry is a protocol surface. The shape is intentionally minimal so it doesn't lock you into a protocol family — A2A, MCP, ACP, plain HTTPS, whatever comes next.
- **Card per entry, optional.** Each entry can point at a protocol-native card if the caller wants the full schema. Or not — the `url` alone is enough for a caller that already knows the protocol.

The catalog is the *index*; the per-protocol cards are the *content*. Same separation-of-concerns logic as the previous post's resolution-vs-description split.

<br>

## Where Pluralism Has to Live

A single card forces a choice the world doesn't make. Real agents speak more than one protocol, and the moment you serve them as one card per protocol you've implied a hierarchy — this one's canonical, that one's the fallback. The catalog refuses that. A2A, MCP, plain HTTPS sit as peer entries in one array, and a caller that only speaks MCP scans for `protocol: "mcp"`, takes the URL, and ignores the rest. The protocol diversity is real; the wrapper is the one layer honest enough to enumerate it instead of ranking it.

The temptation is to make that wrapper smart — have it translate between protocols, normalize the surfaces, hide the differences. ai-catalog stays dumb on purpose. It enumerates and nothing else, which is why a single-protocol caller from two years from now can still read it. A translating wrapper would have to understand every protocol it bridged, and would rot the day a new one shipped. The dumb index is the thing that lasts.

<br>

## What It Leaves on the Table

You still need to *find* the catalog.

`/.well-known/ai-catalog.json` is great once you're talking to the right host. But hosts are domains, and domains aren't agents. If the caller has the domain `darknetian.com` and wants to know *which agents live here*, the catalog spec is silent. The agent has its host; the host has its catalog; the caller still needs to know the host.

This is the exact same gap the previous post's agent-card spec had, scaled up one level. The catalog doesn't do discovery any more than the card does.

DNS-AID slots in below the catalog:

- The walkable namespace `_agents.<zone>` tells the caller what agent hostnames exist under a domain.
- The SVCB ServiceMode record at each agent name pins endpoint hints (ALPN, port) *and* the `well-known` SvcParamKey, which carries the catalog URL.
- The `cap-sha256` SvcParam carries the SHA-256 of the catalog body — DNSSEC vouches for the SVCB, the SVCB vouches for the catalog, the catalog enumerates the protocols, the catalog vouches (via its own entries' optional `card` fields) for the protocol-specific cards. End-to-end integrity, no central registry.

You could imagine the catalog file living at `/.well-known/agent-card.json` instead — the agent-card spec already allows that — and the SVCB pointing at the simpler card directly. That works for single-protocol agents. For multi-protocol agents, the catalog is the layer where the pluralism actually shows up.

<br>

## The Combined Picture

Three records and three files, all integrity-bound:
```text

bookings.darknetian.com.   SVCB 1 endpoint.darknetian.com. (
                                  alpn=h2
                                  bap=a2a,mcp,https
                                  port=443
                                  well-known=/.well-known/ai-catalog.json
                                  cap-sha256=<sha256 of ai-catalog.json>
                                  policy=...
                                  realm=...
                                )

      ↓ resolves to host serving:

      /.well-known/ai-catalog.json
        {
          "catalog": [
            { "protocol": "a2a",   "card": "/.well-known/agent-card/bookings.a2a.json", "url": ... },
            { "protocol": "mcp",   "card": "/.well-known/agent-card/bookings.mcp.json", "url": ... },
            { "protocol": "https", "url": ... }
          ]
        }

      ↓ each "card" entry resolves to the per-protocol agent-card JSON
```

Each layer keeps its own job. DNS-AID does the resolution. ai-catalog does the enumeration. The per-protocol card does the description. The agent runtime does the work.
