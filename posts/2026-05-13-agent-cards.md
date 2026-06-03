---
layout: content.njk
contentType: blog
title: Agent Cards — The Well-Known JSON
description: agent-card.github.io standardizes how an agent describes itself at /.well-known/. DNS-AID resolves names; agent cards describe what answers at those names.
date: 2026-05-13
tags: [dns-aid, ai agents, agent cards, well-known, identity]
hero: ./src/assets/img/posts/agent-cards/hero.png
heroAlt: Rendering of a /.well-known/agent-card/bookings.json document in neon-on-dark monospace, showing the load-bearing fields — id, provider, description, an interfaces[] array of three protocol entries (a2a, mcp, https), capabilities, securitySchemes, version — laid out like an ASCII-art card
list: yes
---

Anyone who's tried to wire two agent runtimes together — Claude Code talking to an MCP server, an A2A planner calling a remote skill — has run into the same boring problem twice. What *is* this thing? What can it do? What endpoint do I hit? What does it want for auth?

The answer that's coalescing around the open web is the *agent card*. It's an [RFC 8615 well-known JSON document](https://www.rfc-editor.org/rfc/rfc8615) served from `/.well-known/agent-card/<name>.json` (or, depending on the variant, just `/.well-known/agent-card.json` at the agent's host root). The shape is standardized at [agent-card.github.io](https://agent-card.github.io/ai-catalog/#identity) and adopted by both [A2A](https://a2aproject.dev/) and [MCP](https://modelcontextprotocol.io/).

It's a small thing. It's also doing a remarkable amount of work that nobody quite acknowledges.

<br>

## What the Card Actually Carries

Strip the spec down to the fields that do the work and you get something like:
```json

{
  "id": "darknetian-bookings",
  "name": "bookings",
  "description": "Book a 30-minute consult.",
  "provider": {
    "organization": "Darknetian",
    "url": "https://darknetian.com",
    "contact": "https://darknetian.com/about/"
  },
  "interfaces": [
    {
      "protocol": "a2a",
      "transport": "https",
      "url": "https://bookings.darknetian.com/a2a"
    }
  ],
  "capabilities": ["scheduling", "calendar"],
  "securitySchemes": { "...": "..." }
}
```

A few of these fields are doing more than they look:

- **Identity** is a stable string the agent owns (`id`), not a URL that might change if you re-host.
- **Provider** is who's responsible. Operator accountability, baked into the wire format.
- **Interfaces** is a list — same agent, multiple ways to call it. A2A here, MCP there, HTTP JSON for the curious. Each entry pins protocol + transport + URL.
- **Capabilities** is the closest thing in the spec to "what does it do" — declared, not inferred.
- **Security schemes** is the agent saying *how to talk to me securely* before you ever try.

That's it. It's an honest, RFC-8615-shaped, machine-readable business card.

<br>

## Why Self-Description Beats the Alternative

The card makes one move that the framework-config era got backwards: it puts the description on the agent, not on the caller. Before agent cards, every framework rolled its own configuration store. The agent shipped tools; some other layer remembered what those tools meant. That coupling broke the moment the agent moved, was rehosted, or shipped a new skill — the remembered config was now describing an agent that no longer existed. Putting the description on the agent's own host, at a well-known path, lets the agent move and the caller follow. The single source of truth is the thing being described.

That choice rides RFC 8615, which is one of those quietly successful pieces of infrastructure — `security.txt`, `change-password`, `apple-app-site-association`, `openid-configuration` all live in `.well-known/`. Stable path, no out-of-band registry, content-typed, easy to cache. The card inherits all of it for free.

The part that's easy to miss is what the card *refuses* to do. It describes the agent and stops — it never says how a caller should orchestrate it. That refusal is the reason MCP, A2A, and HTTPS-native callers can all consume the same artifact instead of three competing ones. A card that dictated usage would have to pick a runtime, and the moment it picked one it would stop being neutral ground. Description is portable across consumers; orchestration isn't.

<br>

## What It Leaves on the Table

Here's the gap that DNS-AID fills.

The card answers *what* — given you can reach the agent at `bookings.darknetian.com`, here's what to expect. It doesn't answer *where* — given the domain `darknetian.com`, what agents live here? Resolution is out of scope for the card spec, deliberately. The spec authors made the right call: identity format and discovery substrate are different concerns and shouldn't be conflated.

But that means *something else* has to do discovery. Options:

- **Registry.** A central directory the caller queries. Works; introduces a trust dependency the agent operator can't audit. Whoever runs the registry decides who's listed.
- **Curated catalog.** A static list at a well-known location — `/.well-known/ai-catalog.json` (more on that in the next post). Better than a registry; still doesn't answer cross-domain "agents matching X."
- **DNS.** The substrate that already resolves everything else on the internet. Operator-owned, federated by design, integrity-protected via DNSSEC.

DNS-AID is the DNS answer. The walkable `_agents` namespace tells a caller which names exist; the SVCB ServiceMode record at each name points at the endpoint and pins `cap-sha256` to the integrity hash of the card JSON. Card + DNS-AID together is *find* + *describe*. Neither one tries to do the other's job.

<br>

## The Stack Picture

If you draw it out:
```text

Caller
  ↓ resolves
DNS-AID  ────────────────────────  walkable directory + integrity
  ↓ points at
Agent host (any)
  ↓ serves
/.well-known/agent-card/<name>.json  ────  identity + interfaces + caps
  ↓ describes
Agent runtime (Claude / MCP server / A2A planner / your code)
```

Each layer does one thing. Each layer can evolve independently. DNS-AID can grow new SvcParams; the card spec can add new fields; the agent runtime can change tools — none of it requires the other layers to know.

That's the architecture I want for agent discovery. Open at every layer. No mandatory registry. The agent card earns its place by staying inside its lane — describe, and let resolution be someone else's job.
