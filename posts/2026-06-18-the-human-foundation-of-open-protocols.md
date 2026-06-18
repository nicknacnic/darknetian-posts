---
layout: content.njk
contentType: blog
title: The Human Foundation of Open Protocols — Notes From the Domain Summit Keynote
description: "My first keynote, in Berlin: why standards outlive the people who write them, the three schools of thought forming around agentic identity, where domains actually fit, and why the question of who mediates the agentic web is being answered right now."
date: 2026-06-18
tags: [dns-aid, ai agents, ietf, itu, standards, domains]
hero: ./src/assets/img/posts/the-human-foundation-of-open-protocols/hero.jpg
heroAlt: 'Me on stage at Domain Summit 2026 in Berlin, mid-talk, with a slide reading "A quick anecdote about technical standards" over a close-up of railroad track behind me'
list: yes
---
Does anyone know, offhand, the diameter of the Space Launch System's solid rocket booster?

It's 12.17 feet. 370.94 cm. Not ten feet, not four meters — an oddly specific number to build a rocket around. I opened with that question because the answer runs back four thousand years, and nobody planned a single step of it.

The boosters are built in Utah. They launch from Florida. To get from one to the other they ride a train east, and every eastbound train out of Utah passes through the Moffat Tunnel — which is sized a hair wider than the U.S. standard railroad gauge of 4 feet 8.5 inches. That gauge comes from English expatriates who built American railways the way they'd built tramways back home. The tramways inherited it from wagons. The wagons used it to ride the ruts worn into ancient roads. The roads were largely Roman, paved for a two-horse war chariot. And the Romans took *that* spacing from the Mesopotamians, who'd worked out that two horses beat a single ox on cost and speed and the original disaster recovery innovation to more effectively wage war. I'm sure you might even take it further back from an evolutionary biology perspective on why horses are the size they are, if you wanted to.

So the booster that carried Hubble to orbit and built the ISS is, at its widest point, constrained by the back end of two horses from four thousand years ago. (I borrowed that chain from [this old Northeastern course page](https://ece.northeastern.edu/courses/geu110/2003fa/booster.html); like most good engineering parables it's truer in spirit than in every link.) Standards live forever. 

<br>

## Why I Was in the Room at All

For the first time in my career, I was asked to tell a story from *my* perspective. Most of what I present to customers is pre-built — marketing owns the brand, product owns the story, sales leadership knows which asset to deploy when. This time I custom-built the slides, and the content was just my lived experience working in the IETF, the ITU-T, and the agentic discovery space. I'm not *the* expert in any of those rooms. I've just been in enough of them to draw the map for someone who hasn't.

The venue made the point for me. InterNetX — part of the IONOS group — ran [Domain Summit 2026](https://hub.internetx.com/en/domain-summit-2026) out of Fotografiska, a building whose history includes the birth of the company logo, of window shopping, and a chunk of Berlin's graffiti culture. The theme was metamorphosis. Hard to pick a more on-the-nose place to argue about what the next layer of the internet should become.

<br>

## Who's Actually Writing the Rules

Before the agentic part, a detour through how the U.S. shows up — and I am emphatically not a government representative here. Congress reliably does two things: pass the NDAA and handle appropriations. Almost everything else is optional as we are finding out, which is why optional things get bolted onto appropriations as provisions since they are "must pass". A recent one tried to forbid *any* state-level AI regulation for ten years. The House passed it 216–215; the Senate stripped it 99–1.

The takeaway isn't the politics — it's that in the U.S., enterprises, not the government, are going to drive who participates in standards venues. And those venues aren't monolithic. They sort along two axes: how policy-driven versus tribal they are, and how sovereign versus nationalized. The ITU-T is quasi-governmental; the IETF and W3C are quasi-sovereign — "rough consensus and running code"; the Linux Foundation and the Agentic AI Foundation are unaffiliated. Each pulls a different lever.

<br>

## Three Schools of Thought

There's an absolute explosion of agentic standards right now — hundreds of proposals worldwide, enough that the venues themselves are straining under the influx. But the field really splits on one question: *where does an agent's identity actually live?* Answer it three ways and you get three roughly equal-sized camps.

- **Domains.** Identity lives in DNS; DNSSEC and DANE close the trust loop. No app store.
- **Applications.** DNS at most finds the host; identity and state live above it, in HTTPS, a ledger, a registry. The app store.
- **The others.** Wait for the perfect spec, or bet that agentic is a bubble. A new paradigm, or none.

The good news: all three factions show up inside every SDO — but not every SDO brings the expertise or the will to care about a given layer. The IETF is the ancestral home of DNS, so that's where the domains work belongs. The ITU has X.509 and friends; the LF and AAIF live at the application and system level. None of that is a fault. It's just where the gravity is.

<br>

## Where Domains Actually Fit

Here's the part I most wanted the room to hear: **domains aren't the answer. They're an answer.** It takes a village.

A domain is the closest thing the internet has to a sovereign primitive — global uniqueness enforced by a long tail of hundreds of registry operators who all agree on a universal takedown process and back each other up when something breaks. It's also the natural integration point for the two parties who actually transact: consumers usually know where they want to go, and enterprises have trusted business relationships. The domain is the key that unlocks the handshake between them.

It's also the *first* place an agent emits intent about what it wants to do — which means the existing control plane (protective DNS, defensive registrations, lookalike detection) becomes defense-in-depth for free. But the headline benefit isn't security. It's that the consumer gets a choice in *how* their agents consume content.

<br>

## A Small Part of the Answer

DNS-AID is not my retirement plan or a bid for world domination. It deliberately solves a small part of the problem — the part that needs a name the whole internet already trusts. It resolves a name to an agent; it doesn't catalog every agent you run, and it doesn't replace MCP or A2A.

The mechanic that matters: DNS carries the anchor and a hash, not the payload.
```text

bookings.darknetian.com.  SVCB 1 endpoint.
  bap="mcp,a2a"                    ; how to speak to it
  well-known="/.../bookings.json"  ; where the full card lives
  cap-sha256="MNavz…"              ; a hash that pins those bytes
```

The agent card on the other end — skills, examples, auth, the whole manifest — lives off-DNS over HTTPS, where it can grow freely. The `cap-sha256` makes those bytes tamper-evident. If you're going to ask Claude to book a meeting with me, you already know which agent you want; you don't need to re-download the manifest to confirm it. Small, signed, cacheable in DNS; everything heavy where it belongs. And the service-binding parameters are nearly infinitely extensible per application owner — enroll URIs for Zero Trust proxies, mesh URNs for accelerated WAN, cache hints, MCP-gateway task signals — all of it before I ever shake hands with the application owner.

<br>

## The 1984 Problem

I closed on a worry, because I think it's the actual stake.

In *1984*, the Party suppresses language so the people can't articulate their own oppression. Now picture the near future where nobody visits sites directly — your agent reads the web on your behalf. If that agent quietly couldn't reach content that conflicted with the policies of whatever third party mediated it, and you never knew, would the outcome be any different? Would the internet change in a way that's imperceptible and literally unspeakable?

That's why I don't think it's acceptable for a single third-party app store to sit in the middle of every agentic interaction. It would make your business a function of your relationship with that operator. That's too much power in one place — and the choice of how to consume content is the deliverable I care most about protecting.

<br>

## Get In the Room

After meeting in Shenzhen in March, the IETF cleared a new mailing list — DAWN — to run a BOF in Vienna in July. I'll be there presenting updates to the draft and the DNS-AID SDK, and running a hackathon around our integration with the EU digital-identity wallet ecosystem. In parallel the ITU is spinning up an AI incubation focus group. Think of the arc like X.509: the ITU defined it, but it didn't explode until the Linux Foundation stewarded OpenSSL. There's a piece of agentic identity that domains and network folks are uniquely suited to carry — and the draft is still being written.

After the keynote I sat on a panel about what agentic discovery *should* look like. We disagreed productively, which is the point. Standards live forever; the least we can do is argue about them while we still can.

The full deck — 40 slides from Berlin — is below. And if any of this is wrong, I'd genuinely like to hear why — that's how the work gets better.

{% slidesCarousel "nw-ids", "Domain Summit keynote slide" %}
