---
layout: content.njk
contentType: blog
title: Five Fake Agents on a Real Cloudflare Zone
description: Publishing 5 DNS-AID agent records to darknetian.com — flat primary plus walkable AliasMode, DANE TLSA from throwaway self-signed certs, all DNSSEC-signed end-to-end. No agents actually exist behind any of them.
date: 2026-05-14
tags: [dns, dns-aid, ai agents, cloudflare, dane, hot rfc]
hero: ./src/assets/img/posts/five-fake-agents-real-dns/hero.png
heroAlt: Terminal screenshot of dig output showing SVCB ServiceMode, AliasMode, TLSA, and TXT records resolving with DNSSEC AD flag set
heroVariant: contain
list: yes
---

# Introduction

Every DNS-AID post I've written this year has gestured at a deployment story without showing one. [DCV](https://www.darknetian.com/posts/2026-05-08-dns-aid-dcv/) explained a trust primitive. The [EDNS(0) agent-hint](https://www.darknetian.com/posts/2026-05-12-edns-agent-hint/) post talked about a signaling channel that doesn't have a hint-aware resolver to read it yet. The TXT-fallback PR I shipped this week is real code, but in the post I described it the demo was hand-crafted records on a testbed BIND inside my LAN. Each of those was useful as a piece of a story; none of them showed *what a zone looks like when you treat DNS-AID as a thing you put on your real public domain*.

So I did that. I picked five agents that vaguely resemble services I might one day offer through the Darknetian brand — search across the site, a bookings flow for consults, a threat-intel lookup, a DNS-posture audit, and a security-incident coordinator I named `morpheus` because if it's going to be fictional I might as well have fun — wrote a small Cloudflare-API script, and let it publish twenty-two records to `darknetian.com`. Then I let DNSSEC sign the lot.

None of these agents serve traffic. There is no live endpoint behind `endpoint.darknetian.com`, which I pointed at `10.42.0.42` — a perfectly respectable RFC 1918 address that nothing in this universe will ever connect back to. The TLSA records pin the public keys of self-signed certs I generated in memory and threw out before the script finished, so even if someone tried to handshake, they would have no matching key. What *does* resolve is the substrate: the agent records, the index, the capability descriptors at `https://darknetian.com/.well-known/agent-card/<name>.json`, and — as a small upgrade from a draft version of this post — the `cap-sha256` in each SVCB record actually matches the SHA-256 of the JSON bytes the well-known URL serves. End-to-end integrity is provable from `dig` to `curl` to `sha256sum`.

That's enough to make a public demo land for the audience I care about: people who want to look at a real Cloudflare zone, query it from `1.1.1.1`, and see DNS-AID records the way a consumer agent would see them.

<br>

## The Zone

Twenty-two records, written in one pass:

```text

  1  A          endpoint.darknetian.com            10.42.0.42
 5x  SVCB       <agent>.darknetian.com             priority=1, target=endpoint, alpn, port, bap, well-known, cap-sha256, policy, realm, mandatory
 5x  SVCB       <agent>._agents.darknetian.com     priority=0, target=<agent>.darknetian.com.
 5x  TXT        <agent>._agents.darknetian.com     capabilities=..., version=..., description=..., category=...
 5x  TLSA       _443._tcp.<agent>.darknetian.com   3 1 1 <SPKI SHA-256 of throwaway cert>
  1  TXT        _index._agents.darknetian.com      "agents=name1:proto,name2:proto,..."
```

The shape follows the current draft of <span class="text-neon">draft-mozleywilliams-dnsop-dnsaid-02</span>: the primary owner is a flat FQDN (`search.darknetian.com`), and a walkable alias lives at `<agent>._agents.darknetian.com` in SVCB AliasMode. The walkable form is what a directory or an enumeration crawler hits; the flat form is the canonical owner that everything else points at. The organization index lives at `_index._agents.darknetian.com` — the underscore on `_index` is preserved in `-02`'s IANA underscored-node-names registration alongside `_agents` itself.

<br>

## The Agents

| Name | Protocol | Realm | Category |
|---|---|---|---|
| search | mcp | demo | research |
| bookings | a2a | demo | business |
| threat-intel | mcp | demo | security |
| dns-audit | mcp | demo | security |
| morpheus | a2a | regulated | security |

Each one has a one-line description in the companion TXT record so `dig TXT search._agents.darknetian.com` returns something readable for someone exploring the zone with a terminal. `morpheus` carries a `regulated` realm to show that realm is per-agent, not zone-wide — and also carries an off-spec SvcParamKey that nobody outside this post should recognise. More on that below.

<br>

## One Agent End-to-End

`search.darknetian.com` is the canonical owner; querying it for its SVCB ServiceMode record via `dnspython` against `1.1.1.1` (because macOS-bundled `dig 9.10` doesn't know the SVCB mnemonic and silently downgrades to type A) returns:

```text

search.darknetian.com.  SVCB  1 endpoint.darknetian.com.
                              mandatory="alpn,port"
                              alpn="h2"
                              bap="mcp"
                              port="443"
                              key65400="https://darknetian.com/.well-known/agent-card/search.json"
                              key65401="MNavzKWrM5ccLxObaEb-Wa1uWTZ90fcNa-tNNleD78E"
                              key65403="https://darknetian.com/policy/agents/demo"
                              key65404="demo"
```

The walkable AliasMode at `search._agents.darknetian.com` points back at it (priority 0, no SvcParams), the TLSA at `_443._tcp.search.darknetian.com` pins the throwaway cert's SPKI, and the companion TXT carries the human-readable metadata:

```text

search._agents.darknetian.com.       SVCB  0 search.darknetian.com.
_443._tcp.search.darknetian.com.     TLSA  3 1 1 093f7dfdad15dbc7c74ec9083c9fce308cda2dbe4c67f7a92800f74b2b307d48
search._agents.darknetian.com.       TXT   "capabilities=search,retrieval,project-discovery" "version=1.0.0" "description=Search across darknetian.com posts, project pages, and demos." "category=research"
```

Every one of these answers came back with the AD bit set. `1.1.1.1` is a validating recursive, `darknetian.com` is DNSSEC-signed, the zone uses algorithm 13 (ECDSAP256SHA256), and every record above has an RRSIG sitting next to it.

<br>

## All Five, Pretty-Printed

The full set, pulled live from `1.1.1.1` via a `dnspython` one-liner — the actual `to_text()` output is the SVCB presentation format from RFC 9460 §2.2:

```text

search.darknetian.com.        SVCB  1 endpoint.darknetian.com. mandatory="alpn,port" alpn="h2" bap="mcp" port="443" key65400="https://darknetian.com/.well-known/agent-card/search.json" key65401="MNavzKWrM5ccLxObaEb-Wa1uWTZ90fcNa-tNNleD78E" key65403="https://darknetian.com/policy/agents/demo" key65404="demo"
bookings.darknetian.com.      SVCB  1 endpoint.darknetian.com. mandatory="alpn,port" alpn="h2" bap="a2a" port="443" key65400="https://darknetian.com/.well-known/agent-card/bookings.json" key65401="5G-fai_0_oujRSkryLzUOo9cM2mufrjGsLtlv_HXdwo" key65403="https://darknetian.com/policy/agents/demo" key65404="demo"
threat-intel.darknetian.com.  SVCB  1 endpoint.darknetian.com. mandatory="alpn,port" alpn="h2" bap="mcp" port="443" key65400="https://darknetian.com/.well-known/agent-card/threat-intel.json" key65401="rkyIu3IA2oR9qeKBIMXuGMNwoV8eM65YnxLIaviJC6c" key65403="https://darknetian.com/policy/agents/demo" key65404="demo"
dns-audit.darknetian.com.     SVCB  1 endpoint.darknetian.com. mandatory="alpn,port" alpn="h2" bap="mcp" port="443" key65400="https://darknetian.com/.well-known/agent-card/dns-audit.json" key65401="B0lyzb0Mc1zoDTc79pyGLryrtnj27VX5s7UrX0l1FOU" key65403="https://darknetian.com/policy/agents/demo" key65404="demo"
morpheus.darknetian.com.      SVCB  1 endpoint.darknetian.com. mandatory="alpn,port" alpn="h2" bap="a2a" port="443" key65400="https://darknetian.com/.well-known/agent-card/morpheus.json" key65401="jqUW3GVXwoVBt1jWCmJaeukLlJwkXdbCbPPM3WPpS_Q" key65403="https://darknetian.com/policy/agents/regulated" key65404="regulated" key65500="nebuchadnezzar"
```

Pick `morpheus` out of that list. It carries an extra <span class="text-neon">key65500="nebuchadnezzar"</span> at the end of its SvcParam set that none of the other four have. That key is not defined anywhere in the DNS-AID draft. Section 8 of RFC 9460 says a consumer that doesn't recognise a SvcParamKey must ignore it — so a generic SVCB client doesn't blink. But an operator-aware client that *does* know the key can act on it. See the off-spec callout below.

The walkable aliases all point home:

```text

search._agents.darknetian.com.        SVCB  0 search.darknetian.com.
bookings._agents.darknetian.com.      SVCB  0 bookings.darknetian.com.
threat-intel._agents.darknetian.com.  SVCB  0 threat-intel.darknetian.com.
dns-audit._agents.darknetian.com.     SVCB  0 dns-audit.darknetian.com.
morpheus._agents.darknetian.com.      SVCB  0 morpheus.darknetian.com.
```

And the TLSA fingerprints — five distinct SPKI SHA-256 hashes, one per throwaway cert:

```text

_443._tcp.search.darknetian.com.        TLSA  3 1 1 093f7dfdad15dbc7c74ec9083c9fce308cda2dbe4c67f7a92800f74b2b307d48
_443._tcp.bookings.darknetian.com.      TLSA  3 1 1 1fa8d75c9a45eebd35cdcc2009598cada644f17d9d851037348ca0ffcf7fa596
_443._tcp.threat-intel.darknetian.com.  TLSA  3 1 1 633c0d744579cbc2e596bee43bab8cae8929eb364a63c417e4989f5ff2e97169
_443._tcp.dns-audit.darknetian.com.     TLSA  3 1 1 38367045f8b6f26ba039988037406b3f4fc1c17ea4d43096d872be2a5c0f12fa
_443._tcp.morpheus.darknetian.com.      TLSA  3 1 1 4e6a6437587fef2bfae664f4e784f516be5267ccca28f80ebd65538acfb5d2c5
```

The one-liner that produced the SVCB output:

```python

import dns.resolver
r = dns.resolver.Resolver(); r.nameservers = ['1.1.1.1']
for n in ['search', 'bookings', 'threat-intel', 'dns-audit', 'morpheus']:
    for rd in r.resolve(f'{n}.darknetian.com', 'SVCB'):
        print(f'{n}.darknetian.com.  SVCB  {rd.to_text()}')
```

That's it. `pip install dnspython` if you don't already have it; the `to_text()` method does the RFC 9460 presentation-format rendering for free.

<br>

## An Off-Spec Key on Morpheus

`morpheus.darknetian.com` carries one SvcParamKey the other four agents don't: <span class="text-neon">key65500="nebuchadnezzar"</span>. That key is not in `draft-mozleywilliams-dnsop-dnsaid`. It sits well outside the 65400–65408 range that dns-aid-core uses for its registered private-use SVCB params (`well-known`, `cap-sha256`, `bap`, `policy`, `realm`, `sig`, `connect-class`, `connect-meta`, `enroll-uri` under the draft-02 naming), and there's no SvcParam registration anywhere — public or private — that would tell a consumer what it means.

That is the point. RFC 9460 §8 spells out what a consumer must do with a SvcParam it doesn't recognise:

<div class="terminal-card p-3 my-4"><strong class="h-mono">RFC 9460 §8 — IGNORING UNKNOWN KEYS</strong><br><br>A SVCB-aware client that doesn't recognise a SvcParamKey ignores it. The record is still usable; the unknown key just doesn't influence client behaviour. The only exception is when the unknown key is listed in <code>mandatory=</code>, in which case the consumer must skip the record entirely.</div>

`morpheus`'s `key65500` is *not* listed in `mandatory=` (the mandatory list is still just `alpn,port`), so a generic SVCB client resolves `morpheus.darknetian.com`, treats the unknown key as a curiosity, and uses the record exactly the same way it uses every other ServiceMode SVCB on the zone. Nothing breaks.

But if an operator-aware client — one that has been told the key65500 → `nebuchadnezzar` mapping out of band — knows what `nebuchadnezzar` means, it can act on the metadata. Maybe `nebuchadnezzar` is an authorization token the agent expects on every inbound call. Maybe it's a vessel-routing tag that names which `mcp-server` instance fields the call. Maybe it's a deployment-class hint used by an internal directory. In this demo the value doesn't actually mean anything — but the *mechanism* of carrying operator-private metadata through the public DNS substrate, opaque to consumers who don't need it, is real and useful.

Two reasons this matters for DNS-AID specifically. First, it's a clean demonstration that a deployment can extend the SVCB SvcParam set without coordinating with the IETF and without breaking interop. Second, it sidesteps the recurring objection that "DNS is too structured for arbitrary metadata" — RFC 9460's unknown-key handling is exactly the extensibility hook that lets a publisher attach what they need without forking the spec.

The teardown matcher in the publish script catches this record by FQDN substring, so when the demo zone is torn down, the off-spec key goes with everything else.

<br>

## DANE Without a Real Cert

The TLSA records pin SPKI fingerprints (selector 1, matching type 1 — SHA-256 of SubjectPublicKeyInfo, per RFC 6698 §2.1) from self-signed RSA-2048 certs the publish script generated in memory. Here is the entire cert lifecycle:

```python

key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
cert = build_self_signed(key, common_name=flat_fqdn)
der = cert.public_bytes(Encoding.DER)
spki = load_der_x509_certificate(der).public_key().public_bytes(
    encoding=Encoding.DER,
    format=PublicFormat.SubjectPublicKeyInfo,
)
fingerprint = hashlib.sha256(spki).hexdigest()
# cert + key go out of scope here; only the fingerprint hits DNS
```

This is fine for a DNS-only demo because the TLS handshake never happens. A real DANE-EE deployment would keep the private key in an HSM or at least a config file the actual service can read on startup; here the script literally cannot reproduce the same fingerprint twice because the RSA generation isn't seeded.

For the demo what matters is that the TLSA records look real from the wire — usage 3 (DANE-EE), selector 1, matching type 1, and a properly-shaped 32-byte SHA-256 digest in the data field — and that <mark>a future-PR DANE-aware client would happily refuse to talk to anything whose presented cert doesn't match</mark>. Which, since no presented cert exists, is a delightfully tautological way to be DANE-strict.

<br>

## SVCB in -02 Shape

Direction matters here. The current `-02` draft places the primary owner at the flat name and uses the walkable `_agents` namespace as an AliasMode pointer back to it. That's the inverse of what someone unfamiliar with the draft might guess — they often start out drafting the `_agents` location as the canonical record and the flat name as the alias. I tripped on that myself when scoping the demo; the user-facing logic for AliasMode points *to* canonical, not *from* canonical.

What I ended up with:

```text

search.darknetian.com.            SVCB 1 endpoint.darknetian.com.   # priority 1 = ServiceMode (the real thing)
search._agents.darknetian.com.    SVCB 0 search.darknetian.com.     # priority 0 = AliasMode (the pointer)
```

A consumer that wants to enumerate the zone walks `_index._agents.darknetian.com` for the list of agent names, then queries each `<name>._agents.darknetian.com`, follows the AliasMode pointer to the flat name, and pulls the ServiceMode SVCB with all the metadata. A consumer that already knows the agent name skips the walkable form entirely and queries `<name>.darknetian.com` directly. Same zone, two entry points, one canonical record.

<br>

## Agent Cards That Actually Resolve

The `key65400` value on each SVCB is a URL to a JSON capability descriptor (the `well-known` SvcParam in the draft-02 naming); the `key65401` next to it is the SHA-256 of the bytes that URL is supposed to serve (`cap-sha256`). In the first iteration of this demo those JSON documents didn't exist — the URLs 404'd and the hashes were synthetic placeholders. That's a perfectly defensible shortcut for a "look how the DNS records look" post, but it leaves a piece of the DNS-AID story on the floor: the *point* of `cap-sha256` is that a consumer can dereference the URL, hash the bytes it gets back, and compare to what DNS told them. End-to-end integrity, anchored in DNSSEC at one end and a content hash at the other.

So in the rewrite I generated five real agent cards and parked them at `https://darknetian.com/.well-known/agent-card/<name>.json` — passed through by Eleventy, served as static content by Azure SWA, with `/.well-known/*` carved out of the SPA navigation-fallback rewrite. Four follow the [Model Context Protocol](https://modelcontextprotocol.io/) agent-card shape (`search`, `threat-intel`, `dns-audit`); one is MCP and one is in the [A2A AgentCard](https://a2aproject.dev/) shape (`bookings`, `morpheus`). Each one is a plausible-looking card: capabilities or skills, an auth scheme, input/output modes, a documentation URL pointing back at this post.

The verification chain a consumer would run:

```bash

# 1. Pull the well-known URI from DNS (SvcParamKey 65400)
$ python3 -c "
import dns.resolver
r = dns.resolver.Resolver(); r.nameservers=['1.1.1.1']
for rd in r.resolve('search.darknetian.com', 'SVCB'):
    print(rd.to_text())
" | grep -oE 'key65400=\"[^\"]+\"' 
key65400="https://darknetian.com/.well-known/agent-card/search.json"

# 2. Pull the expected hash from DNS (also signed; AD bit set)
$ dig @1.1.1.1 +short search.darknetian.com TYPE64 | tr ' ' '\n' | grep ^key65401
key65401="MNavzKWrM5ccLxObaEb-Wa1uWTZ90fcNa-tNNleD78E"

# 3. Fetch the JSON and hash the bytes
$ curl -fsS https://darknetian.com/.well-known/agent-card/search.json \
    | openssl dgst -sha256 -binary \
    | openssl base64 \
    | tr '+/' '-_' | tr -d '='
MNavzKWrM5ccLxObaEb-Wa1uWTZ90fcNa-tNNleD78E
```

The base64url-encoded SHA-256 of the JSON byte stream matches the `key65401` value the zone published. Same equality holds for all five agents.

Two things were worth getting right in the JSON itself. First, the `morpheus.json` card carries a small `extensions` block:

```json

"extensions": {
  "darknetian:realm": "regulated",
  "darknetian:vessel": "nebuchadnezzar",
  "darknetian:vessel-note": "The vessel field carries the operator routing token published in DNS as the off-spec SvcParamKey key65500..."
}
```

The `darknetian:vessel` value matches the off-spec `key65500="nebuchadnezzar"` on the SVCB record — same data, two transports. A consumer that doesn't speak the Darknetian operator namespace ignores both; one that does can cross-check that the DNS-layer routing tag and the application-layer card agree.

Second, the cards are explicitly demo cards. Each one has a `notes` field that says so and links back to this post. None of these skills/tools serve real work, but the card *shape* is faithful to what a working MCP or A2A agent's descriptor looks like in 2026, including `protocolVersion`, capability flags, and a documented skills/tools array with examples.

<br>

## Try It Yourself

This is a public zone now. Anyone can run these:

```bash

# Org index — what agents does darknetian.com advertise?
dig @1.1.1.1 +short _index._agents.darknetian.com TXT

# Pick one agent, get its endpoint
dig @1.1.1.1 +short search.darknetian.com TYPE64

# Verify it's DANE-pinned
dig @1.1.1.1 +short _443._tcp.search.darknetian.com TYPE52

# Confirm DNSSEC signed end-to-end (look for "ad" in flags)
dig @1.1.1.1 +dnssec search.darknetian.com TYPE64 | grep flags
```

`TYPE64` and `TYPE52` are the numeric codes for SVCB and TLSA. macOS-bundled `dig 9.10` doesn't know the mnemonics and silently downgrades to type A if you use the string names. `brew install bind` gets you a current `dig` that does, but the numerics work from any version.

The records will stay live indefinitely. If you're noodling on an agent-discovery client and want a real DNSSEC-signed zone to develop against without standing up your own, this is one.

<br>

## What I Learned

Three things came out of it.

One, Cloudflare's API handles RFC 9460 SVCB cleanly, including private-use SvcParamKeys in the `key65400`–`key65408` range. That wasn't obvious in advance. Some managed-DNS providers reject anything outside the standard SvcParamKey set; Cloudflare just stores what you send.

Two, DNSSEC-signing a non-trivial zone with TLSA, SVCB, and structured TXT records was uneventful. ECDSAP256SHA256 over five agents' worth of records adds milliseconds of resolver overhead. The complaints about DNSSEC-deters-adoption that I see in DNS-AID review threads tend to come from people who haven't tried it recently on a modern resolver and a modern managed-DNS provider.

Three, having a real public demo zone is qualitatively different from a testbed BIND zone for the talks I give. It lets the audience reach for their own `dig`, run a query, and see the answer come back from a recursive they didn't have to configure. Every standards conversation I've been in about DNS-AID has been improved by being able to say "go look at darknetian.com" — a reviewer who can resolve the records on their own resolver argues from what the zone actually does, not from what the draft claims.

<br>

## P.S. — Watching the Zone From the Homelab

Five hours after publishing the records, I wanted to know what was actually resolving them. Cloudflare's free tier doesn't ship Logpush (the per-request firehose lives behind Enterprise), but `dnsAnalyticsAdaptiveGroups` on the GraphQL Analytics API is free and bucketed at 1-minute granularity — plenty for a homelab dashboard. So I wrote a 5-minute systemd timer on the [homelab Graylog VM](https://github.com/nicknacnic/graylog) that pulls per-`(queryName, sourceIP)` aggregates, maps each `queryName` back to one of the five agent buckets, and emits the result as GELF into a dedicated Cloudflare index set.

<br>

{% image "src/assets/img/posts/five-fake-agents-real-dns/graylog-agents-dashboard.png", "Graylog dashboard 'Agents' page showing 100 total agent queries, 10 unique recursive resolvers, 9 distinct agent records, 0 NXDOMAINs, plus a per-agent table (bookings 30, morpheus 30, search 20, dns-audit 10) and donut charts for query mix by agent and by type (SVCB 60%, A 20%, TLSA 20%)", "prose-img-lg" %}

<br>

One DNS-AID-specific gotcha worth knowing if you build something similar: "unique visitor count" doesn't really exist on the authoritative side. The `sourceIP` field Cloudflare hands back is the recursive resolver — `1.1.1.1`, `8.8.8.8`, `9.9.9.9`, and so on — not the end user behind it. So the "Unique resolvers" tile in the screenshot is exactly that: how many distinct recursives have asked Cloudflare about these names since the zone went live. It's a useful relative signal of how widely the records have spread through resolver caches, but not a literal user count. Anything finer than that would require either a recursive-side log feed (which DNS-AID's [agent-hint EDNS work](https://www.darknetian.com/posts/2026-05-12-edns-agent-hint/) is partly setting up the affordances for) or a TLS-terminating endpoint behind the SVCB target — both of which the demo zone deliberately doesn't have.

The full poller, index-set config, and dashboard JSON are in the [`graylog`](https://github.com/nicknacnic/graylog) repo if you want to copy the shape — `pollers/cloudflare_poller.py` + `dashboards/cloudflare.py` is the agents-page substrate.
