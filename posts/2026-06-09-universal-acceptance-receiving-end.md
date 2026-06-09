---
layout: content.njk
contentType: blog
title: "ICANN's Universal Acceptance From the Receiving End: Three Internationalized Domain Names"
description: I registered three internationalized domains and tried to make them serve web pages and receive mail on a free stack. DNS did its job every time.
date: 2026-06-09
tags: [dns, universal acceptance, idn, email, infoblox]
hero: ./src/assets/img/posts/universal-acceptance-receiving-end/hero.png
heroAlt: A name flowing through DNS that accepts it, into an application layer that refuses it
heroVariant: contain
list: yes
---

The first working session I sat through at ICANN87 in Seville (my first ICANN meeting in person) was about Universal Acceptance, and the pitch is genuinely moving. An internet where someone's name in their own script works everywhere a Latin one does: their domain, their email, the form field at the bank, the login box. The company I work for, Infoblox, among other things builds lookalike-domain detections. This machinery flags an `xn--`-disguised домен before it phishes someone, and I've spent enough time around that problem to walk into the room sympathetic and skeptical at once. Skeptical won. Not because the idea is wrong. Because the place it breaks is the one place nobody can be ordered to fix.

So I went back to the hotel and tested it on myself.

<br>

## Prerequisites

I registered three internationalized domains, pointed them at my existing site, and tried to receive mail at them. Every single failure was *above* DNS. The root zone, the registry tables, the resolvers, the SVCB and A records — all of that handled non-ASCII names without a complaint. What refused were the applications sitting on top: a static-site host that won't accept a punycode custom domain, a mail layer that won't speak the internationalized protocol, a sender-reputation gauntlet that doesn't care what alphabet you use.

Underneath the plumbing sits a harder problem that no amount of buy-in solves: telling whether two names are "the same" is tractable somewhat in Cyrillic for substitutions and very nearly impossible in Chinese or a sentence / intent based looklike (eg zarabotay-v-gugle = make money in google). The detection problem I've watched up close runs out of road exactly where Universal Acceptance needs it most — so let's start there.

<br>

## The Part That Doesn't Scale — Cyrillic Is a Lookup, Chinese Is a Guess

The danger Universal Acceptance opens is the homograph: `даркнетиан` versus a Latin-mixed impostor, the lookalike that phishes. For Cyrillic-versus-Latin this is a *solved-shaped* problem. Unicode publishes [`confusables.txt`](https://www.unicode.org/reports/tr39/) (UTS #39); you reduce each string to a "skeleton" of canonical shapes and compare. It's a deterministic table lookup — hard at scale, but bounded. You can list the confusable set and check membership.

Chinese is not that problem. Two strings can be different in Unicode, identical in meaning, and identical in how a reader perceives them — Simplified and Traditional forms of the same word, semantic variants, characters a human treats as interchangeable that no glyph-similarity table relates. There's no finite confusables set to consult; "the same name" becomes a *nearest-neighbor* question in a space where the metric is meaning, and meaning is what software is worst at. ICANN knows this — it's why the [IDN Variant Program](https://www.icann.org/resources/pages/idn-2012-02-25-en) and the Root Zone Label Generation Rules exist, convening per-script panels to hand-author variant relationships precisely because you cannot compute them.

So the asymmetry is structural, and it cuts the wrong way. Cyrillic — small alphabet, clean confusable table — is the script where defense is cleanest. The CJK billions, the population that would most expand an accessible internet, sit in the script where "is this a lookalike of that?" has no crisp answer. Cyrillic is somewhat policeable with a table. Chinese forces you to approximate a human judgment about intent, and approximations are where attackers live. More adoption doesn't dissolve that. It sharpens it.

<br>

## The Setup — Three Names That Shouldn't Be Hard

To watch the plumbing actually behave, I picked one name in three scripts of increasing obscurity, all under `.com` (which Verisign supports IDN registration for across hundreds of language tables):

| Script | Name | A-label |
|---|---|---|
| Cyrillic | `даркнетиан.com` | `xn--80aakfpkvdvx.com` |
| Han | `暗网人.com` | `xn--gmqq59cupr.com` |
| Thaana | `ހނދ.com` | `xn--hqbe1a.com` |

Those three aren't the same trick — they walk straight up the difficulty gradient from the section above. The Cyrillic `даркнетиан` is a letter-for-letter transliteration (`d-a-r-k-n-e-t-i-a-n` → `д-а-р-к-н-е-т-и-а-н`): the classic homograph, and the *easy* case — a deterministic character substitution a confusables table catches.

The Han `暗网人` is a different beast: a semantic calque, not a sound-map.

| Char | Pinyin | Meaning |
|---|---|---|
| 暗 | àn | dark / dim / hidden / secret |
| 网 | wǎng | net / web / network |
| 人 | rén | person / human |

`暗` + `网` is `暗网`, the actual Chinese word for *darknet*; I added `人` (person) for the *-ian* — as in Martian, human, the personhood suffix. Nothing relates `暗网人` to `darknetian` except *meaning*.

Thaana `ހނދ`? It's neither a translation nor a transliteration — three bare consonants of the Maldivian alphabet (*haa-noonu-dhaalu*, no vowels). Picked for the glyphs, not the meaning.

Registered at a mainstream registrar, nameservers delegated to Cloudflare... that part took ten minutes and zero drama — which is the point. The supply side, the part ICANN actually contracts, works.

<br>

## Step One — Using ICANN's Own Tool Against Myself

Before building anything I wanted the honest answer to "is my mail even ready for this?" There's a clean way to check: ICANN ships an [EAI support survey tool](https://www.icann.org/ua) — a Java surveyor that walks a domain's MX servers and asks whether they speak `SMTPUTF8`, the extension that lets the *mailbox* half of an address (left of the `@`) carry non-ASCII characters. I pointed its core probe at my own MX and a couple of others.

The probe is dead simple: connect to the MX, `EHLO`, look for `SMTPUTF8` in the capability list, then try a `MAIL FROM` with an internationalized address.

```text

$ probe darknetian.com   # MX → iCloud
250-8BITMIME
250-STARTTLS
... no SMTPUTF8

$ probe infoblox.com     # MX → Proofpoint
250-8BITMIME
... no SMTPUTF8

$ probe gmail.com
250-8BITMIME
250 SMTPUTF8             # ← the only yes
```

A scoreboard of who can actually *receive* an internationalized mailbox:

| Provider | Domain | SMTPUTF8 |
|---|---|---|
| iCloud | darknetian.com | no |
| Proofpoint | infoblox.com | no |
| Gmail | gmail.com | yes |

My personal domain runs on iCloud; my employer's runs on Proofpoint. Neither can take a `δοκιμή@`-style mailbox. They advertise `8BITMIME` — UTF-8 in the message *body* — but not `SMTPUTF8`, the envelope-address piece UA actually needs. So the fully-internationalized email address, the headline feature of UA, is unavailable to me at the receiving end no matter what I register. Gmail was the lone yes.

<br>

<div class="terminal-card p-3 my-4"><strong class="h-mono">THE USABLE SUBSET</strong><br><br>What does work everywhere is an ASCII mailbox on an internationalized domain: nic@даркнетиан.com. The local-part stays ASCII; the domain travels as punycode, which is plain ASCII on the wire, so no SMTPUTF8 is required. That's the slice of UA you can actually ship today — and it's the slice I built.</div>

<br>

## Step Two — Making Them Serve (Azure Says No to Punycode)

My site is a static app on Azure Static Web Apps. The obvious move — add each IDN domain as a custom domain so Azure issues a cert — dies on contact: Azure Static Web Apps **does not accept punycode custom domains at all.** Not a tier limit, a flat refusal.

Fine, let Cloudflare terminate TLS and proxy to the origin. That needs an Origin Rule to rewrite the `Host` header to the app's real hostname — and on Cloudflare's free plan that override is a paid entitlement:

```text

"not entitled to use the HostHeader override"
```

Two product walls, neither a DNS problem. The way through was a small reverse-proxy Worker — free, and its own `fetch()` to the origin handles the upstream TLS, which sidesteps the question of the zone's SSL mode entirely:

```js

export default {
  async fetch(request) {
    const MAIN = "icy-wave-00bdf5a10.4.azurestaticapps.net";
    const ANS  = "wonderful-pond-08c02b810.7.azurestaticapps.net";
    const url = new URL(request.url);
    const origin = url.hostname.startsWith("ans.") ? ANS : MAIN;
    url.hostname = origin;
    const headers = new Headers(request.headers);
    headers.set("Host", origin);
    return fetch(url.toString(), { method: request.method, headers, redirect: "manual" });
  }
};
```

Bind that to `<idn>/*`, `www.<idn>/*`, and `ans.<idn>/*` on each zone, point the records at a proxied placeholder, and Cloudflare's Universal SSL covers the A-labels for free. All three names now serve the site over HTTPS, the Unicode form sitting right there in the address bar.

<br>

## Step Three — Making Mail Arrive (Two 550s and a Reputation Wall)

Receiving was the easy half conceptually: Cloudflare Email Routing is free, supports IDN domains, and forwards `nic@<idn>` to my real inbox. Records in, rule in, destination verified. Done.

Then I tried to *send* a test message to those addresses from a laptop, and met the reputation gauntlet head-on. First wall:

```text

550 Sender IP reverse lookup rejected (2620:f:8000:210:...)
```

No reverse DNS on the sending IP. Force IPv4, which has a PTR, and the second wall lands:

```text

550 5.7.26 Cannot forward emails that are not authenticated.
```

The receiver won't relay unauthenticated mail — no aligned SPF or DKIM, no delivery. This is the real reason "just run your own mail server" is dead advice for an individual: the protocol works, but a cold IP with no reputation and no auth gets refused before the content matters. The fix wasn't technical — I sent it from an account that *is* authenticated, a normal mailbox, and it sailed through Cloudflare's forwarder into my inbox:


{% image "src/assets/img/posts/universal-acceptance-receiving-end/inbox.png", "An email to nic@даркнетиан.com rendered in Cyrillic, forwarded into a normal inbox", "prose-img-md" %}


`nic@даркнетиан.com`, in Cyrillic, in a mainstream mail client, delivered. The minimal-footprint UA mailbox is real — it just routes through a managed forwarder, never a box I run.

<br>

## Who Can Actually Be Made to Accept

Walk back up the stack and notice *where* every wall stood. Registry IDN tables: ICANN's house, contracted, working. Registrar selling the name: contracted, working. Root and resolvers: working. Then the failures — Azure refusing punycode, a SaaS gating a header rewrite, a mailbox provider skipping `SMTPUTF8`, a receiver refusing unauthenticated relay — all live in the application and operator layer, which ICANN has zero contractual leverage over. It can mandate the *sellers*. It can only *ask* the people who must *accept*.

That gap isn't new, but the institutional signal got louder: ICANN [wound down the Universal Acceptance Steering Group in 2025](https://uasg.tech/) and folded the work under a President's Committee. Read it however you like; the structural fact is unchanged. You have a body that can write the supply side into a contract, and a demand side — every framework author, form validator, mail vendor, and cloud product on earth — that answers to no contract at all. UA isn't stalled because the standards are missing. They shipped years ago. It's stalled because acceptance is a verb performed by ten thousand parties who were never promised anything and owe nobody.

<div class="terminal-card p-3 my-4"><strong class="h-mono">AN ON-BRAND POSTSCRIPT</strong><br><br>I posted this writeup on Bluesky and the ASCII URL <code>darknetian.com</code> linkified — the IDN siblings <code>даркнетиан.com</code> / <code>暗网人.com</code> / <code>ހނދ.com</code> rendered as plain text. The same gap, one stack up. Two community PRs are already in flight: <a href="https://github.com/bluesky-social/atproto/pull/4156">bluesky-social/atproto#4156</a> rewrites the rich-text URL regex against the WHATWG URL spec so non-ASCII URIs validate, and <a href="https://github.com/bluesky-social/social-app/pull/7308">bluesky-social/social-app#7308</a> teaches the client to render IDN handles in their Unicode form with the right sanitization rails. Neither merged yet at time of writing. This is exactly the shape the post is about — protocol fine, implementations catching up one PR at a time, no contract that compels.</div>

<br>

## Backing the Contact Page — the Names, in a TXT Record

There's one more place these names belong, and it's a nice closing proof that the DNS layer never flinched. My contact page runs a small animation off TXT records at `nic.darknetian.com` — `Org`, `Title`, `Email`. The owner name there is plain ASCII, so there's nothing to punycode on the *left* side; the internationalized addresses go in the record *value*. And that's legal without ceremony: TXT rdata is a sequence of 8-bit octets ([RFC 1035](https://www.rfc-editor.org/rfc/rfc1035) §3.3.14), so UTF-8 in the value is RFC-compliant. The punycode-versus-Unicode question doesn't even arise — a TXT value isn't a hostname, it's bytes.

So I can publish the IDN addresses literally:

```text

nic.darknetian.com.  300  IN  TXT  "Email-IDN = nic@даркнетиан.com, nic@暗网人.com, nic@ހނދ.com"
```

`dig` hands the non-ASCII bytes back as decimal escapes by default — same data, wire-faithful presentation:

```text

nic.darknetian.com. 300 IN TXT "Email-IDN = nic@\208\180\208\176\209\128\208\186\208\189\208\181\209\130\208\184\208\176\208\189.com, ..."
```

Modern resolvers and clients decode the UTF-8 fine; the escapes are just `dig` being conservative about what it prints. In the zone manifest my reconcile agent pushes to Cloudflare, it's one more record object:

```json

{
  "name": "nic.darknetian.com",
  "type": "TXT",
  "content": "Email-IDN = nic@даркнетиан.com, nic@暗网人.com, nic@ހނދ.com",
  "ttl": 300,
  "proxied": false,
}
```

<br>

## What I Took Back From Seville

The DNS already lets the next billion users have their name. I watched it do so three times in an afternoon without a hitch. The rest of the stack still half-refuses to *read* that name — and the half that's hardest to fix isn't a missing RFC, it's that nobody can be compelled to accept, and that even when they try, "is this name a dangerous twin of that one?" is a question we can answer in Cyrillic and only guess at in Chinese, and those are just the languages that were top of mind for me. The permutation complexity here is unbelievable and I'm very grateful to Infoblox's threat intel team for picking the hard fights.

I left more impressed by the mission and less optimistic about the timeline. An accessible internet is the right fight. It just doesn't get won in the root — it gets won, or lost, in every input box that does or doesn't believe a name like `даркнетиан.com` is real.
