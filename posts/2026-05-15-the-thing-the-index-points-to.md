---
layout: content.njk
contentType: blog
title: The Thing the Index Points To
description: DNS-AID's path-2 index leaf names a registry the draft explicitly leaves out of scope. Wiring ANS — a registration authority plus transparency log — to be that registry.
date: 2026-05-15
tags: [dns, dns-aid, ai agents, transparency log, ans, azure, cloudflare]
hero: ./src/assets/img/posts/the-thing-the-index-points-to/hero.png
heroAlt: Terminal screenshot showing a signed C2SP checkpoint emitted by the live ans-tl deployment at ans.darknetian.com, alongside the SVCB record at _index._agents.darknetian.com pointing back at it
heroVariant: contain
list: yes
---

# Introduction

The [DNS-AID](https://www.darknetian.com/project/2026-05-12-dns-aid/) work I've been writing about names three discovery states a requesting agent can be in. The first — knows the agent and the domain — is the one a single SVCB lookup solves, and is the one the [five-fake-agents](https://www.darknetian.com/posts/2026-05-14-five-fake-agents-real-dns/) zone exercises end-to-end. The second — knows the domain but not which agent — is where it gets interesting. The leaf is in DNS at `_index._agents.<domain>`, but what the leaf points *to* is left as an exercise. Could be a TXT. Could be a JSON file at a well-known URI. Could be a hosted registry with its own trust model. The draft is deliberate about leaving the format open, because the registry is upper-layer work and the IETF avoids over-specifying upper layers that will diverge across deployments.

That ambiguity is a seam. The five-fake-agents post filled it with a one-line TXT — `agents=search:mcp,bookings:a2a,…` — which works for five agents and falls apart at fifty. Useful for a smoke test; not useful as a trust anchor.

I wanted to see whether the [Agent Name Service (ANS)](https://github.com/godaddy/ans), the registration-authority-plus-transparency-log I've researched in parallel, could *be* the thing the leaf points to. ANS is already producing every byte such a registry needs — signed event envelopes, SCITT COSE receipts, C2SP signed-note checkpoints, X.509 identity and server certs pinned through TLSA. The question was whether you could plug them together without re-doing either spec.

This post walks the feasibility check, the local end-to-end test of the integrity chain, and the production wire shape now serving at `ans.darknetian.com`.

<br>

## ANS, the Short Version

[ANS](https://github.com/godaddy/ans) is two Go binaries (`ans-ra` for the registration authority, `ans-tl` for the transparency log) plus a verifier (`ans-verify`) and a dev DNS server (`ans-dns`). The RA mints identity certs and server certs for agents, runs domain control validation via ACME or DNS challenges, and produces signed event envelopes. The TL appends each event to a Tessera-backed Merkle tree, mints a SCITT COSE_Sign1 receipt with the inclusion proof, and emits a [C2SP signed-note checkpoint](https://c2sp.org/signed-note) on a 2-second cadence.

The pieces that matter for the path-2 question:

- **Per-agent event envelope** carrying `ansName` (canonical), `agentHost` (flat), `endpoints[]`, plus an `attestations` block with `identityCerts[]`, `serverCerts[]`, and `dnsRecordsProvisioned[]` — the actual TXT and TLSA records the agent expects to find in DNS, attested to be the ones the RA verified.
- **SCITT receipt** with `leafHash + leafIndex + path[] + rootHash + treeSize`, signed by the TL's ECDSA P-256 key, advertised at `/root-keys` in the [sumdb-note](https://research.swtch.com/tlog#signed_note) verifier-key format.
- **Read-side HTTP API** that's almost entirely cacheable: `/checkpoint`, `/root-keys`, `/tile/{level}/{index}`, `/v1/agents/{id}`, `/v1/agents/{id}/receipt`. Every one of those is GET-only and immutable per event.

That last point is the trick. A transparency log's read side is naturally static — the tile bytes never change once written, the checkpoint advances monotonically, and per-agent receipts only change when the agent itself re-registers or renews. You don't need a running database to serve them; you need *a file tree*.

<br>

## Feasibility, On Paper

Before doing any plumbing I wrote a feasibility plan asking three questions. They're not all the questions, but they're the ones whose answers would either kill the idea or commit me to it.

**Does the wire shape fit?** Draft-02's path-2 leaf points "to a registry." That can be a SVCB record with an HTTPS target carrying a `well-known` URI parameter, which is exactly how every other DNS-AID record in the zone works. So the path-2 SVCB at `_index._agents.darknetian.com` lands on `ans.darknetian.com`, the `well-known` URI is `/.well-known/agents-index.json`, the `cap-sha256` parameter pins the bytes of that JSON. Same chain-of-hashes pattern that anchors per-agent records. No new ceremony.

**What does the JSON body look like?** [dns-aid-core](https://github.com/infobloxopen/dns-aid-core) — the reference DNS-AID client library — already has an `fetch_http_index` path with a documented JSON shape and a `parse_http_index` parser. I could either invent my own envelope or emit the shape dns-aid-core already understands. The latter is obviously the right answer.

**What does ANS need to expose that it doesn't today?** Three small gaps. One: a "list agents in this zone" read route, which can actually be derived from the RA's existing `GET /v2/ans/agents?status=ALL` for a single-tenant deployment. Two: a snapshot envelope that binds the listing to a specific TL state via the checkpoint (so a consumer can verify the listing was current at `treeSize=N`). Three: name-shape mapping between ANS canonical names (`ans://v1.0.0.search.darknetian.com`) and the flat DNS-AID owner name the consumer needs to query in path 1.

That's it. The integrity chain — RA-attested cert fingerprint becomes TLSA bytes in the zone — falls out for free, because the RA already publishes the SHA-256 SPKI fingerprint on `serverCerts[]` and DNS-AID's TLSA usage-3 selector-1 matching-type-1 format wants exactly those bytes.

<br>

## Local Test, Five Fake Agents Again

I brought up `ans-ra` and `ans-tl` locally on my laptop and registered the same five hosts from the [Cloudflare zone post](https://www.darknetian.com/posts/2026-05-14-five-fake-agents-real-dns/) — `search`, `bookings`, `threat-intel`, `dns-audit`, `morpheus`, all under `darknetian.com`. Each one walked the full RA lifecycle: register, verify-acme, verify-dns, ACTIVE. The TL ended at `treeSize=5` with five SCITT receipts.

Then I wrote a small Python adapter that pulls each agent's state from the RA, the badge plus receipt from the TL, and emits two JSON files: one in the dns-aid-core HTTP-index shape, and one enriched envelope carrying the SCITT proof bits for direct verification:

```bash

$ python3 build_index.py
wrote /tmp/ans-darknetian-index/agents-index.json (5 agents)
wrote /tmp/ans-darknetian-index/ans-index-enriched.json
checkpoint treeSize: 5
```

The integrity cross-check is what I most wanted to see. The TL attests the SHA-256 of each agent's server-cert SPKI on `serverCerts[0].fingerprint`. The DNS zone publishes that same hash as the rdata of a TLSA record. If the chain holds, those two strings are byte-for-byte equal:

```text

morpheus      zone_tlsa=64419a8ed21f68850d58c5a45b55478405791fd3b49c770aa1a0f782936d940f
              badge_fp =64419a8ed21f68850d58c5a45b55478405791fd3b49c770aa1a0f782936d940f
dns-audit     zone_tlsa=3de0a0b21c969ffa779e489ae1b65b445c9cc15f1686b21bcce28fd69753dbe4
              badge_fp =3de0a0b21c969ffa779e489ae1b65b445c9cc15f1686b21bcce28fd69753dbe4
threat-intel  zone_tlsa=91ba07170a816de3ae434325648a5d4e21867b2acf28460d608ed134b5304371
              badge_fp =91ba07170a816de3ae434325648a5d4e21867b2acf28460d608ed134b5304371
bookings      zone_tlsa=4e3c90b3f620274ab1e523aa0175f4e98e340e186fd869b6403a7a6539649d2c
              badge_fp =4e3c90b3f620274ab1e523aa0175f4e98e340e186fd869b6403a7a6539649d2c
search        zone_tlsa=4ad8f481b4b87b335644bca6b924d614432f5d15483ce774edea83db829af179
              badge_fp =4ad8f481b4b87b335644bca6b924d614432f5d15483ce774edea83db829af179
```

That's the chain-of-hashes that anchors every path-2 entry to its path-1 record. The TL signs the cert fingerprint; the cert fingerprint becomes the TLSA bytes; the consumer can walk from a path-2 listing to a per-agent TLSA verification without ever trusting the listing endpoint beyond DNSSEC and TLS.

The dns-aid-core round-trip went the way it should:

```bash

$ uv run python probe_dns_aid_core.py
parsed 5 agents from ANS-built index
  morpheus       fqdn=morpheus.darknetian.com     proto=mcp  endpoint=https://morpheus.darknetian.com/mcp
  dns-audit      fqdn=dns-audit.darknetian.com    proto=mcp  endpoint=https://dns-audit.darknetian.com/mcp
  threat-intel   fqdn=threat-intel.darknetian.com proto=mcp  endpoint=https://threat-intel.darknetian.com/mcp
  bookings       fqdn=bookings.darknetian.com     proto=mcp  endpoint=https://bookings.darknetian.com/mcp
  search         fqdn=search.darknetian.com       proto=mcp  endpoint=https://search.darknetian.com/mcp
```

`dns-aid-core`'s own parser accepting the ANS-derived JSON without a single field rename was the moment the feasibility question collapsed from "could this work" to "this works."

The fifth check was offline receipt verification:

```bash

$ for id in $(curl -s -H "Authorization: Bearer ans-dev-key-change-me" \
    "http://localhost:18080/v2/ans/agents?status=ALL" | jq -r '.items[].agentId'); do
    bin/ans-verify -agent "$id"
  done | grep '✓'
✓ receipt leafHash: 91de5de5738ebc725801e1cb8009f6fd4bacf55fba3bf09ec3f99bea44218c4b
✓ receipt leafHash: 6143fee5907e72a4f0055ed14eaca0a81f2aeeb7cd75f6830e441c578c9964b2
✓ receipt leafHash: bff6169f0ccbae47d64a625421738fc984dedcbaefecedf05954732a151d440f
✓ receipt leafHash: 4de5194fea56259be32628f9355f05929f0fdec8d53f4698275f0ab2b9a8da15
✓ receipt leafHash: 4e21da2d355acb4f4eb27b806be48cddc76a87c39ba7f4583e93e1164d71fa37
```

Each leaf hash matches the `merkleProof.leafHash` on the corresponding badge, which is the same hash a consumer derives from the JCS-canonical event bytes the RA originally signed. The whole chain — RA signs payload bytes → TL appends the leaf → SCITT receipt carries inclusion proof → checkpoint roots the tree — verifies offline against a single pinned ECDSA P-256 public key from `/root-keys`.

That was enough to commit.

<br>

## The Wire Shape That Emerged

The path-2 SVCB I ended up with:

```text

_index._agents.darknetian.com.  SVCB 1 ans.darknetian.com.
                                      alpn="h2"
                                      key65400="https://ans.darknetian.com/.well-known/agents-index.json"
                                      key65401="<sha256-b64url of the JSON body>"
```

Same `well-known` / `cap-sha256` pair the per-agent records use, just one level up. A consumer that walks path 2 dereferences the `well-known` URI, hashes the response bytes, compares to `cap-sha256`, and only then trusts the listing. The chain-of-hashes property of the zone is preserved at the org-index level the same way it is at the per-agent level.

The trick is that `cap-sha256` over a live transparency log is a tension. The log is by definition append-only and growing; if the SVCB's hash is over the listing JSON, then every registration or revocation rotates the SVCB. Two honest options here:

- **Snapshot-pinned.** Re-emit the SVCB with a new `cap-sha256` every time the index body changes. The chain holds; the cost is zone churn on every mint.
- **Drop cap-sha256 on the index entry.** Rely on DNSSEC + TLS + the embedded checkpoint signature inside the JSON for trust. Cleaner ops; one less knot in the chain.

I picked option one. The CI publisher computes `sha256(agents-index.json)` after every mint, writes it into `dns/manifest.json`, and Cloudflare's record gets PATCHed in the same deploy. The cost is real but bounded — for a darknetian-scale zone with maybe a dozen agents max, the SVCB rotates a dozen times a year. Acceptable.

<br>

## Publishing Shape

A transparency log's read side is naturally static. The tile bytes never change once written, the checkpoint advances monotonically, and per-agent receipts only change when the agent itself re-registers or renews. So the public side of `ans.darknetian.com` is a flat directory of bytes — `root-keys`, `checkpoint`, the tile tree, badges, receipts, `agents-index.json` — exported from a local `ans-tl` after each mint and served straight off Azure Static Web Apps.

Two properties of this shape matter more than the hosting choice.

**The private key never leaves the laptop.** The RA and TL signing keys live in a gitignored directory on my machine. The public site is a derivation, never a writer. A common alternative is to host the log as a long-running service with its signing key in cloud KMS, but that makes the cloud operator a trust boundary — if the KMS account is compromised, the log can be backdated. Keeping the private key local means the worst the cloud can do is serve stale or absent records; it cannot sign new ones.

**CI is dumb on purpose.** GitHub Actions never runs `ans-ra` or `ans-tl`. It validates that the published snapshot's `cap-sha256` matches the manifest the laptop committed, reconciles a small `dns/manifest.json` against Cloudflare via the Cloudflare API, and uploads the snapshot to SWA. Three steps, no state, no keys. If the CI runner is compromised, the worst it can do is corrupt the public site — it cannot mint or revoke an agent.

<br>

## The Inversion People Try First

When I described this pipeline to a colleague, they suggested the obvious-looking complication: "If somebody adds an agent record directly in Cloudflare, CI should notice and import it into the TL." That intuition is wrong, and it's wrong in a way worth spelling out.

A transparency log derives its value from being the source of truth for what has been signed and when. If you treat DNS as the writer and the TL as the secondary store, you've inverted the trust relationship — anyone who can edit DNS can effectively backdate the log, because the log is now just mirroring DNS state instead of attesting to its own. The whole point of having a TL behind the path-2 index is that the listing is anchored to something stronger than the zone editor's permissions.

So the CI check is not a *sync*; it's a *drift detector*. The nightly `drift-check.yml` reads every Cloudflare DNS record tagged with the comment `managed-by-darknetian-ans`, compares it byte-for-byte against `dns/manifest.json`, and fails the run on any divergence. If somebody edits a record in the CF dashboard, the next morning's run is red and I know about it. The remediation is to re-run the deploy workflow, which re-asserts the manifest. DNS doesn't get to be the writer.

Unmanaged records — anything in the zone without the marker comment — are out of scope. That's how the five-fake-agents records from last week's post coexist with the ANS-managed records: they don't carry the tag, the publisher refuses to touch them, the drift checker ignores them. Adoption is opt-in record by record.

<br>

## What's Live Today

`ans.darknetian.com` is bound, signed, and serving the static tree out of Azure SWA, fronted by a Cloudflare-managed Microsoft cert. The path-2 SVCB at Cloudflare is published and resolving against `1.1.1.1`:

```bash

$ dig +short TYPE64 _index._agents.darknetian.com @1.1.1.1
\# 136 000103616E730A6461726B6E657469616E03636F6D00000100030268
32FF78003868747470733A2F2F616E732E6461726B6E657469616E2E
636F6D2F2E77656C6C2D6B6E6F776E2F6167656E74732D696E646578
2E6A736F6EFF79002B796A30574F367346553447436369595542576A
7A76766671724268383639646F654F433250703545493159
```

That's the raw wire format because dig 9.10 doesn't decode SVCB. The presentation form decodes to a ServiceMode record targeting `ans.darknetian.com`, with `alpn=h2`, a `well-known` pointer at `/.well-known/agents-index.json`, and the `cap-sha256` over the empty-log JSON body.

The TL itself is signing and serving at `https://ans.darknetian.com/checkpoint`:

```text

darknetian-ans
0
47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=

— darknetian-ans E/C1ZMEiPtaM+RlFxlDq1ENy5V8wRgh9NZwXHgXvQgAEaju/iNuX5+WcfF5X0nfHQ8icEdFYbupFnjvtA6SLrpDSdoU=
— darknetian-ans E/C1ZGV5SmhiR2NpT2lKRlV6STFOaUlzSW10cFpDSTZJakV6WmpCaU5UWTBJaXdpZEhsd0lqb2lTbGRVSWl3aWRHbHRaWE4wWVcxd0lqb3hOemM0T0Rnek5qQTJmUS5leUpqYUdWamEzQnZhVzUwUm05eWJXRjBJam9pWXpKemNDOTJNU0lzSW05eWFXZHBiaUk2SW1SaGNtdHVaWFJwWVc0dFlXNXpJaXdpY205dmRFaGhjMmdpT2lJME4wUkZVWEJxT0VoQ1UyRXJMMVJKYlZjck5VcERaWFZSWlZKcmJUVk9UWEJLVjFwSE0yaFRkVVpWUFNJc0luUnBiV1Z6ZEdGdGNDSTZNVGMzT0RnNE16WXdOaXdpZEhKbFpYTnBlbVVpT2pCOS5IWmU5YVBuNGN2cnc0eGZrWFVrQ250UHpnWVkwWUs5VUdlcG9GQzdCQ3VtSlFZa0VrQlVqVEowMHgyZEkxWDNYWVdQYnc2cVBkYXZDVWhETll2SXQ2dw==
```

`darknetian-ans` is the origin string my laptop's TL was bootstrapped with. The first line after the origin is the tree size (currently `0` — no agents minted to the production log yet). The third line is the base64 of the empty Merkle root, which is the SHA-256 of the empty byte string — a perfectly respectable cryptographic identity for "I have not yet attested anything." The two dash-prefixed lines are the C2SP signed-note signatures: the primary one signed by the TL's ECDSA P-256 key, the second one the JWS additional-signer carrying the same checkpoint as a JWT for verifiers that prefer that format.

The verifier key the consumer pins, served at `https://ans.darknetian.com/root-keys`:

```text

darknetian-ans+13f0b564+AjBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABFKDFsltSn5djkiS3i8QfBCGY+moDeRb9dg+Kpw0d/PzzxrDrxw7QFZHjpLItvlWvqirWmMpoAS4Qgs0I2GKQw0=
```

That's `<origin>+<keyhash-hex>+<base64(0x02 || SPKI-DER)>` in sumdb-note format. The `0x02` prefix byte declares the algorithm as Ed25519/ECDSA-flavored signed-note (per the sumdb-note spec). The keyhash (`13f0b564`) is what gets matched against the kid byte on receipts so the verifier can pick the right key in O(1). Once you've pinned that line, every checkpoint signature and every SCITT receipt for the rest of the log's life verifies against it offline.

And the index body itself, at `https://ans.darknetian.com/.well-known/agents-index.json`, with the snapshot envelope I describe in the next section:

```json

{
  "origin": "darknetian-ans",
  "treeSize": 0,
  "rootHash": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=",
  "checkpoint": "darknetian-ans\n0\n47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=\n\n— darknetian-ans E/C1ZME...",
  "agents": {}
}
```

`agents: {}` is honest empty. Everything around it is the verifiable context.

<br>

## What Just Changed — Wrapping the Index in a Checkpoint Envelope

The first cut of the path-2 body was the bare dns-aid-core map: `{}` while empty, `{name: {...}}` once an agent's in. The `cap-sha256` in the SVCB pinned the bytes of that body, which gave you "the JSON bytes were what the publisher said they were when this SVCB record was last written." That's a weaker integrity story than the per-agent records get — the per-agent SCITT receipts anchor to a Merkle tree rooted in a signed checkpoint. There's no reason the org-index shouldn't, too.

So the publisher now wraps the agents map in an envelope carrying the C2SP checkpoint, the tree size, and the root hash:

```json

{
  "origin": "darknetian-ans",
  "treeSize": 0,
  "rootHash": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=",
  "checkpoint": "darknetian-ans\n0\n47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=\n\n— darknetian-ans E/C1ZMEi…\n— darknetian-ans E/C1ZGV5…",
  "agents": {}
}
```

The diff against the previous body, before and after the publisher change:

```diff

-{}
+{
+  "origin": "darknetian-ans",
+  "treeSize": 0,
+  "rootHash": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=",
+  "checkpoint": "darknetian-ans\n0\n47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=\n\n— darknetian-ans E/C1ZMEi…",
+  "agents": {}
+}
```

The change is one helper function in `export-snapshot.py` and one line in `build-manifest.py`. The CI deploy step that recomputes `sha256(agents-index.json)` and compares it against `dns/manifest.json` caught the hash change automatically, the publisher PATCHed the SVCB at Cloudflare with the new `cap-sha256`, and the next dig pulled back the updated record:

```text

# cap-sha256 before:  yj0WO6sFU4GCciYUBWjzvvfqrBh869doeOC2Pp5EI1Y
# cap-sha256 after:   81dsIpuHKT9fpPvVv4jJ7kit2P_b7E445rwMO_49lmc

$ dig +short TYPE64 _index._agents.darknetian.com @1.1.1.1
\# 136 000103616E730A6461726B6E657469616E03636F6D00000100030268
32FF78003868747470733A2F2F616E732E6461726B6E657469616E2E
636F6D2F2E77656C6C2D6B6E6F776E2F6167656E74732D696E646578
2E6A736F6EFF79002B38316473497075484B5439667050765676346A
4A376B697432505F62374534343572774D4F5F34396C6D63
```

The hash near the end of that hex dump — `38316473497075484B5439667050765676346A4A376B69…` — decodes ASCII to `81dsIpuHKT9fpPvVv4jJ7kit2…`, which is the new `cap-sha256` over the wrapped body. The DNS-layer pin tracked the body change without manual intervention.

The bigger property this buys: a consumer can now verify the index is *consistent with a specific TL state*, not just "what the publisher said." Pin `/root-keys` once, verify the `checkpoint` signature inside the envelope against that key, and you know the listing was current at exactly the tree state the checkpoint signed. Subsequent per-agent receipts the consumer fetches by `agentId` will each carry inclusion proofs against the same tree — if any of those proofs claim a tree size larger than the index's checkpoint claimed, the consumer can refuse them. That's the elevation from "trust the listing endpoint" to "trust the same key that signs every other artifact in the log."

<br>

## What's Next

The bigger conceptual win is one I hadn't expected when I started: <mark>the path-2 leaf is the natural place for a domain to advertise an ANS instance, and an ANS instance is the natural backing for a path-2 leaf.</mark> We kept the registry out of scope in the draft because it's upper-layer work and the deployments will diverge, but the *coupling* between the DNS leaf and the registry behind it is tight enough that in practice the layering dissolves. Once a domain runs an ANS, the path-2 record writes itself. Once a consumer can verify the path-2 listing, it gets path-1 verification for free via the same chain-of-hashes that anchored the TLSA bytes to the cert fingerprint to the receipt.

That, I think, is what "complementary upper layer" looks like when you actually build one.

<br>

## P.S. — What the Graylog Dashboard Is Saying

The same [homelab Graylog](https://github.com/nicknacnic/graylog) poller that's been tailing the [five-fake-agents zone](https://www.darknetian.com/posts/2026-05-14-five-fake-agents-real-dns/) for the last day picked up the new infrastructure as soon as the first SVCB resolved. The Agents page rolls every per-agent query name into a `cf_dns_agent` bucket, and in the trailing 24 hours one bucket is dominating:

<br>

{% image "src/assets/img/posts/the-thing-the-index-points-to/graylog-ans-queries.png", "Graylog 'Agents' dashboard tab showing Total agent queries (24h) = 60, Unique resolvers (24h) = 5, Distinct agent records (24h) = 1, NXDOMAINs on agents = 0; per-agent breakdown table showing the 'ans' bucket with 60 queries, 5 unique resolvers, 3 query types, 1 distinct record", "prose-img-lg" %}

<br>

60 queries against the `ans` bucket, 5 unique recursive resolvers fanning out across the path-2 leaf and the `ans.darknetian.com` gateway, 3 distinct query types (SVCB at the leaf, A and TYPE65 chasing the gateway CNAME), and zero NXDOMAINs. The five-fake-agents path-1 records are still in the zone and resolvers will still ask about them, but in the trailing 24h window the share of *agent-classified* queries flowing through the path-2 record set is substantially higher than what's hitting the older path-1 names — a real signal that the path-2 leaf, once it exists, becomes the natural first stop for any consumer that doesn't already know which agent it wants.

The classification logic is the same shape as in the prior post — `cloudflare_poller.py` maps each `queryName` back to an agent bucket and pushes the aggregate as GELF — so the same caveat applies: `sourceIP` is the recursive resolver, not the end user. "5 unique resolvers" is how many distinct recursives have asked Cloudflare about these names, not how many people are running discovery clients against darknetian.com.
