---
layout: content.njk
contentType: blog
title: Agentic Semantic Search Is Hard, But It Doesn't Have To Be
description: Building a more trustworthy search service is possible from existing internet primitives.
date: 2026-08-05
tags: [dns-aid, ai agents, searxng, ans, ard, agent discovery]
hero: ./src/assets/img/posts/agent-search-stack/hero.png
heroAlt: Three-tier network diagram showing twenty agent publishers across the top (anthropic.com, agentshare.dev, darknetian.com and others), five resolver nodes in the middle (DNS-AID zone walk, agent-trust-discovery, AgentFinder federation, SearXNG, enterprise catalog), and ten client types at the bottom, connected by a dense web of crossing lines with no single center point
heroVariant: contain
list: yes
---

Myself and some friends wrote an internet draft, it's about embedding agentic metadata into the DNS. Perhaps you've seen me speak or previously read about [DNS-AID](https://www.darknetian.com/project/2026-05-12-dns-aid/)?

The draft defines 3 general agentic discovery statuses for inter-organization tasks: 
1. Known agent / domain
2. Known agent | domain
3. NOR known agent & domain

DNS is quite good at solving the first two. If I already know I want to speak to agent.example.com, I query it. If I already know I want to speak to a certain trusted provider, I can query a well-known entry point to their organization to find out more, similar to how I might query www.example.com to find their website and find services to pivot from there. We call this a catalog or index. 

The last discovery status is, admittedly, a weakness to DNS. The DNS shouldn't be modified for more trivial enumeration, nor should a specification exist for a multi-record-return type query. The third use case gets at an agent's task intent: "find me an agent that is capable of X." Yet, our co-author group feels strongly the answer to this problem can be composed of the first two primitives DNS solves. 

This blog is the proof of concept that followed.

<div class="terminal-card p-3 my-4"><strong class="h-mono">OPIONATED ASIDE</strong><br><br>A search provider you can't fire isn't a search provider, it's a rent-seeking mediator wearing a different hat. Search should compete on ranking quality (trustworthiness) and freshness, not commercial gamification. The instant it answers a query, you should be able to walk away from it and still reach the agent out-of-band.</div>

<br>

## My Monologue on the DNS

The DNS is composed of three (ish) tiers, and scales without any single point of control because each tier is operated by different entities with different incentives.

**Authoritative servers** are operated by the zone owner and each organization controls its own zone. Nobody else can add or remove records from it. Sovereignty is structural (or in this case, delegated), not a policy promise from a platform.

**Recursive resolvers** sit between clients and authoritatives. They cache answers, absorb query load, and are operated by thousands of different entities — ISPs, enterprise IT, anycast public resolvers, and even homelab instances. No single operator sees all traffic, and most times thanks to minimal responses even middleware observation is limited. If one is unavailable, clients point at another. The redundancy isn't designed in by any individual operator; it emerges from the fact that running a resolver is cheap and the protocol is open.

**Stub resolvers** are local, minimal, and dumb. They ask the recursive resolver. They don't negotiate topology or build indexes. They just ask.

Above all of that: the root. The DNS root is served by 12 independent organizations — universities, nonprofits, a Japanese academic consortium, European internet registry bodies, US military, global standards bodies — operating more than 1,800 anycast nodes across every continent. No single government controls all of them. Any attempt to remove a TLD from the root zone proceeds through ICANN's multi-stakeholder process, is globally visible within minutes, and faces opposition from operators who answer to different legal jurisdictions. Domains can re-delegate between TLDs. Zones can migrate authoritative servers to any provider. Should an adversary attempt to coerce ICANN, or impinge the zone root signing from Verisign, other organizations will take over. The distribution is the protection.

DNS is resistant to the thing AI agent registries are not: compulsion. When governments have issued export control orders restricting access to specific AI model providers, those restrictions operate at the API layer (the place where the call is made). If discovery also runs through a single registry headquartered in one jurisdiction, the restriction can move upstream. Finding the agent becomes the choke point, not just calling it. A government order to a single registry operator removes discoverability with no multi-stakeholder process, no visibility, no recourse for the agent's publisher or the companies that depended on finding it.

There also exists an economic angle... one which I find more troubling. A registry that controls discovery controls commercial advantage: which agents surface for a given query, which are buried, which disappear entirely under pressure from a competitor or a regulator. DNS didn't give any single indexer control over the resolution layer. The indexers competed on ranking quality; the publishers controlled the records. The agentic web needs the same separation — not as a design aspiration, but as an operational reality that exists only if we build it. It must be by choice, not accidental.

<br>

## The Proof of Concept

I'm a bit lucky in the sense that part of my daily tasks include staying up-to-date on emerging agentic standards across many of the organizations innovating in the space. As it turns out, costs to interoperate are the cheapest they've ever been (thanks Claude!). So what happens when we string the ideas together?

**Discovery** I wanted to start with crt.sh, since every TLS certificate ever issued is logged and presumably, a public agent is provided a certificate. Querying crt.sh for agent-shaped SAN patterns like `%.agents.%`, `ans.%`, `agentregistry.%`, `mcp.%` and OAuth authorization servers that expose agent-bound JWKS endpoints.

**Probing** My by no means exhaustive list of checks after candidates surface:

- ARD (`/.well-known/ai-catalog.json`) — multi-protocol catalog with semantic query hints
- ANS (`/.well-known/agents-index.json`) — name service index
- DNS-AID v1 TXT index records and v2 SVCB subdomains — zone-native, DNSSEC-anchored
- A2A agent card (`/.well-known/agent.json`) — per-agent identity and capabilities
- AgentFinder federation (`/search`) — semantic query surface that aggregates across domains without a central index
- `llms.txt` and `ai.txt` — human/machine access policy hints
- `did:web` (`/.well-known/did.json`) — decentralized identity anchoring against a verifiable DID document
- OAuth 2.0 authorization server metadata (`/.well-known/oauth-authorization-server`) — surfaces the JWKS endpoint and token issuer, identifying the cryptographic authority behind the agent's call surface

{% image "src/assets/img/posts/agent-search-stack/atd-ui.png", "A screenshot of the output of ATD", "prose-img-lg" %}

**Indexing** So you've identified agents, now what? Enter [agent-trust-discovery](https://github.com/agentnameservice/agent-trust-discovery) aka ATD: a SQLite+FTS5 database with a five-dimension trust vector (integrity, identity, solvency, behavior, safety) and a REST import and search API. 

**Search** Is trivial with SearXNG wired to a custom engine plugin pointed at ATD. One query box, results carry source format, protocols, capabilities, and trust status. 

{% image "src/assets/img/posts/agent-search-stack/sear.png", "A screenshot of the output of SearXNG ATD search", "prose-img-md" %}

It can't be overstated the revelation that is setting your own trust policy to discover trusted endpoints. Though it's before my time, it feels a lot like what the internet used to be in chaining together trusted networks through your own policy. Huh. A revitalization of how things were.

<br>

## The Trust Tuning

ATD's five-dimension trust vector is the answer to a problem DNS-AID explicitly punts on. The draft's Security Considerations section says it plainly: "Authentic records may point at a malicious or compromised agent. Consumers MUST treat the records published in DNS-AID as a verifiable transport for metadata, not as a trust signal in their own right; trust judgments MUST be made out of band by combining DNS-AID records with reputation, attestation, or organizational policy systems." The trust registry is that out-of-band system.

Even a likely future working group where this draft may get picked up, DAWN, the charter discussions also punt trust to a later recharter.

The dns-aid-core SDK takes this further with a CEL (Common Expression Language) policy engine layered on top of the trust vector. The idea: an organization publishes a policy bundle (a JSON document at the URI carried in the DNS-AID SVCB `policy` SvcParam) and that policy compiles to enforcement at two levels. At Layer 0, domain-based rules compile to RPZ (Response Policy Zone) directives: standard CNAME-based blocking or custom TXT-based DNS-AID actions that fire at the recursive resolver before the call even leaves the network. At Layer 1 and 2, CEL expressions evaluate at runtime against the full PolicyContext: trust score thresholds, caller identity, TLS version, MCP tool name, circuit state, jurisdictional attributes.

What this means for an enterprise: they can write policy rules like:
```text

request.trust_score.identity < 0.7 → deny
request.tls_version == "1.2" → warn
request.target_circuit_state == "open" → deny
```

…and those rules gate which agents are reachable, not just which ones appear in search results. An organization subject to export control obligations can enforce them at the resolver level without depending on any registry operator to do it for them. That's the inverse of the centralization problem: instead of hoping a registry enforces your policy, you run your own resolver and enforce it yourself.

SearXNG's ranking can consume the trust vector directly via surfacing higher-scored agents first and surfacing `dns-only / UNVERIFIED` stubs last with a visual provenance tag. The search layer decides what to surface; the policy layer decides what's reachable; the DNS layer provides the authoritative data. Three different concerns, three different operators, no single party that controls all three.

<br>

## The Syndication Question

The stack as built queries one local registry. The architecture supports more. SearXNG's engine model is an abstraction for what could come next: referer queries. Adding an enterprise catalog engine, a Hugging Face model hub engine, or an AgentFinder federation endpoint is a config entry. Query fan-out and result aggregation happen for free.

The harder question is curation across sources. ATD's trust vector is the proposed answer: different source formats generate different scores, results filter and rank accordingly. The agreement on what trust means, in a way that multiple registries can participate in without any one of them controlling the definition, is the work that remains.

But the shape of the solution is visible. It's the same shape DNS used. Publishers own their records. Resolvers are distributed and replaceable. Caches compete on freshness. Search competes on ranking quality. A domain can migrate between TLDs; an agent can migrate between registries. Those properties exist because the data lives in DNS, not in a platform's database.

<br>

## Roadmap
Today, the dns-aid-sdk allows for an agent to publish its own policy document. It is also enable to emit a policy preference via EDNS(0) to a resolver / search function beyond semantic search (eg "only provide answers that don't allow access to untrusted data"). The dimensionality of how agents will be categorized is ongoing work. 

We are also currently evaluating enabling search to include agents.md, skills.md, and other local / remote origin repositories where teams currently list agents and skills. 

We hope you'll follow along! 😊