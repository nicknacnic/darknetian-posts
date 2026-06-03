---
layout: content.njk
contentType: blog
title: AgentFinder — Federation, Semantic Search, and the DNS Gesture
description: AgentFinder adds representativeQueries and a /search federation API on top of ai-catalog. Its DNS-SVCB gesture is exactly what DNS-AID specifies. They should know about each other.
date: 2026-05-18
tags: [dns-aid, ai agents, agentfinder, ai-catalog, federation, semantic search]
hero: ./src/assets/img/posts/agentfinder/hero.png
heroAlt: Diagram of an AgentFinder federated query — a natural-language query box at the top fans out via parallel /search calls to three domains (darknetian.com matches with score 0.92, example.com returns no match, other.org returns a low-score alternative), and the winning candidate resolves through a DNS-AID SVCB record at bookings._agents.darknetian.com to bookings.darknetian.com
list: yes
---

[AgentFinder](https://github.com/nlweb-ai/AgentFinder) starts from a different question than the agent-card and ai-catalog specs. Those two solved *what is this agent* and *how do I call it*. AgentFinder asks: *how do I find an agent I don't know the name of yet?*

That sounds tautological — surely you have to know what you're looking for to look for it — but it isn't. You might know what *kind* of agent you want ("something that books meetings") without knowing the canonical name ("`bookings`"). You might know the domain you trust ("darknetian.com") without knowing which agents that domain runs. You might want to query a *registry* and get back a ranked list rather than a single deterministic lookup.

AgentFinder is the spec where all three of those questions live. It piggybacks on ai-catalog for the per-agent content shape and adds two things on top: semantic search via `representativeQueries`, and a federation API via `/search`. And — somewhat quietly — it gestures at DNS as the resolution substrate without quite specifying how. That last part is why I keep coming back to it: the gesture is almost word-for-word DNS-AID, and neither spec says so.

<br>

## What AgentFinder Adds

<br>

### representativeQueries

Each agent's catalog entry can carry a list of canonical phrasings the agent expects to handle:
```json

{
  "name": "bookings",
  "representativeQueries": [
    "book me a meeting next Tuesday at 2",
    "what slots are open this week",
    "cancel my consult"
  ],
  "...": "..."
}
```

A registry indexing N agents now has training data for a semantic matcher — an embedding model can map "I want to schedule a call" to the bookings agent without anyone hand-curating intent labels. It's also a forcing function: writing the representative queries makes the operator articulate what the agent is *for* in user-readable terms.

The format doesn't pick an embedding model or a similarity metric. That restraint is correct — those evolve faster than spec text.

### /search Federation API

The other half of AgentFinder is a registry-side HTTP endpoint. Any host can expose `/search` (typically at the catalog's host) that takes a free-text query and returns matching agents:
```http

GET /search?q=book+a+consult&limit=5
Host: darknetian.com

200 OK
{
  "results": [
    { "agent": "bookings", "score": 0.92, "card": "..." },
    { "agent": "scheduling", "score": 0.71, "card": "..." }
  ]
}
```

The federation story: a meta-registry queries N domains' `/search` endpoints in parallel and aggregates. No central index. Each domain owns its agents and its semantic-matching behavior. Cross-domain discovery without giving any one party the keys.

This is *very* much in the spirit of how the rest of the federated web works — RSS, ActivityPub, IndieWeb. Take a known content format, expose a query API at a well-known path, let the network of small participants federate.

<br>

## The Bet AgentFinder Makes

The contestable claim in AgentFinder isn't that semantic search is useful — everyone agrees on that. It's that discovery can stay federated. The default answer the industry reaches for is a central registry: one index, one ranking, one party deciding who's listed and in what order. AgentFinder bets the opposite — that a `/search` endpoint at a well-known path, aggregated across domains the caller already trusts, beats a single authoritative directory. The agents in a registry are the ones the *domain operator* exposes; no third party can list an agent the operator didn't publish, and no third party can bury one it doesn't like.

That bet costs something, and the spec hand-waves the bill. "Anyone can stand up a `/search` endpoint" is true the way "anyone can write an RFC" is true — there's no reference implementation, so participating means re-deriving the wire format from prose. And federation assumes someone already knows *which* domains to ask: the only way to build that list is to crawl a wild fraction of the internet's zones, which is a cost nobody in the spec volunteers to pay. A central registry would at least be consistent and cheap to query. AgentFinder takes the messier, costlier path on purpose — the alternative hands one party the keys to who gets found — but "federated" is doing a lot of unfunded work in that sentence.

<br>

## The DNS Gesture

Here's where AgentFinder gets interesting from a DNS-AID perspective. The spec mentions:

> Discovery via DNS Service Binding (SVCB) at `_catalog._agents.{domain}` and `_search._agents.{domain}` is RECOMMENDED but not required.

Read that twice. AgentFinder *recommends* SVCB records at `_catalog._agents.{domain}` and `_search._agents.{domain}` — names that exist in exactly the underscore-namespace shape DNS-AID specifies. The spec leaves the SvcParam keys, the priority semantics, the integrity hash binding, and the index aggregation pattern *unspecified* — and then says "but you should do this."

DNS-AID does this. The `_agents` namespace, the SVCB ServiceMode/AliasMode split, the `well-known` SvcParamKey carrying the catalog URL, the `cap-sha256` SvcParamKey carrying the integrity hash — all of it is the wire format the AgentFinder gesture leaves to "future work."

So AgentFinder's federation story is incomplete exactly where it matters most. It says discovery should ride DNS, then declines to say what the records carry — which means today the recommended path can't be implemented interoperably. Two implementers reading the same RECOMMENDED line would ship incompatible records. The two specs aren't competing; they're stacked. AgentFinder says *what semantic discovery means*; DNS-AID says *what the SVCB at `_catalog._agents.{domain}` actually carries*. A footnote in either pointing at the other would close the gap that currently makes the gesture unbuildable.

<br>

## The Full Stack

Five layers, no centralization:
```text

Semantic query                          ← user intent
  ↓ AgentFinder /search                ← federation surface
  ↓ ai-catalog.json                    ← multi-protocol enumeration
  ↓ DNS-AID SVCB at _agents.<zone>     ← walkable directory + integrity
  ↓ agent-card.json                    ← per-protocol identity
  ↓ agent runtime                      ← actually answers
```

Five formats, five well-known paths or RR types, no registry vendor in the middle. Each layer is replaceable if better ideas arrive. Each layer is auditable today.

This is the architecture I keep coming back to, and AgentFinder is the layer that finally connects the cataloging story to the resolution story. The DNS gesture in its spec is almost-DNS-AID. The two should adopt each other.
