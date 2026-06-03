---
layout: content.njk
contentType: blog
title: Wiring the CTEM Spiderweb
description: A pipeline that unifies Infoblox CTEM, lookalike-domain monitoring, brand protection, and open-source attack-surface signals into one Graylog dashboard — keyed by finding name, deduped across sources, and tagged with bug-bounty eligibility.
date: 2026-05-17
tags: [easm, ctem, graylog, security, homelab]
hero: ./src/assets/img/posts/ctem-spiderweb/hero.jpg
heroAlt: Three figures in Guy Fawkes masks seated at a curved console of triple monitors lit by green hacker-style UI, framed by an industrial geodesic ceiling structure and red rim lighting (photo by Tima Miroshnichenko on Pexels)
list: yes
---

My investigation workflow lives in Graylog. The [homelab Graylog VM](https://www.darknetian.com/posts/2026-05-07-graylog-deepened/) already ingests iLO, iDRAC, Cradlepoint, Aruba, NIOS, Cloudflare, and a handful of other pollers; when I need to chase something, I open Graylog and pivot from there. The lookup tables, the enrichment pipelines, the IP-to-FQDN resolution, the dashboards I actually use — all of it is already wired up around that one place.

External attack-surface data lives elsewhere by default. Infoblox CTEM has its own console. SOC-Insights lookalike-domain monitoring has its own console. Axur brand protection has its own. Each is good at what it does, and none of them is the place I open when something needs investigating. What I wanted wasn't a replacement — I wanted those signals shaped as GELF events sitting in the same Graylog instance as everything else, so they pivot against client_fqdn lookups, share the same widget grammar, and live alongside the operational telemetry. Then I dogfooded the whole thing against `darknetian.com` plus a small roster of public-company names to give the pipeline a real dataset.

The interesting parts were the dedup key, the "findings on owned-only" rule, and the bounty-eligibility tag.

<br>

## The Sources

Each of these contributes one slice of the picture:

| Source | What It Knows |
|---|---|
| Infoblox CTEM | Exposures on owned assets — DNS hygiene, weak TLS, exposed services |
| Infoblox Lookalike | Newly-registered typosquats, classified as phishing / suspicious / other |
| Axur brand protection | Active phishing kits, credential leaks, infringing content tickets |
| Dossier | Whois / PTR / passive-DNS pivots from a seed indicator |
| Certificate Transparency | New subdomains within minutes of cert issuance |
| subdomain.center | Historical passive-DNS subdomain enumeration |
| VirusTotal | Reputation, sibling domains, communicating samples |
| Built-in dangling-DNS | CNAMEs pointing at unclaimed cloud resources |

Infoblox CSP rides over a private MCP adapter that handles the platform's session quirks; the rest are documented APIs.

<br>

## Expansion Math

Per organization, the pipeline expands seed domains in five stages, in order: Dossier pivots, Certificate Transparency, subdomain.center, VirusTotal subdomains + siblings, and a Wikipedia infobox pass for organizations flagged with M&A history. Each stage logs its contribution:

```text

expand[org-a]: seed=2 research=4 dossier=7 crtsh=183 vt=24 total=220
expand[org-b]: seed=1 research=3 dossier=11 crtsh=412 vt=39 total=466
expand[org-c]: seed=2 research=0 dossier=4 crtsh=58  vt=8  total=72
```

The CT side does most of the heavy lifting — one apex routinely fans out to a few hundred subdomains. Dossier contributes related-apex hints that the public sources miss (cousin brands, acquired-company domains that still resolve). Wikipedia catches the M&A sprawl that nothing else does.

One rule that has saved me from a lot of bad findings: **findings fire on `seed_domains` only.** Everything in the expanded set feeds monitoring — new CTEM seeds, new lookalike targets, new brand assets — but discovered domains don't become subjects of attribution. "I found this domain via Dossier" is a reason to watch it; it is not a reason to blame its problems on the customer.

<br>

## The Dedup Key

Each source emits a finding shape that looks structurally similar but isn't normalized: CTEM calls it `exposure_title`, Axur calls it `ticket_type`, the dangling-DNS detector calls it `kind`. The pipeline normalizes them into one envelope:

```json

{
  "asset": "auth.example.com",
  "finding_name": "Missing SPF Record",
  "severity": "medium",
  "sources": ["infoblox_ctem", "dossier"],
  "bounty_eligible": true,
  "bounty_platform": "hackerone"
}
```

The dedup key is `(asset, finding_name)`, **not** `(source, asset, finding_name)`. When three sources notice the same DMARC gap on the same host, it stays one row with three confirmations in the `sources` array. The widget shows which three. The earlier draft used the wider key and produced a dashboard where the same finding appeared three times in three columns; the right view is one row that says *three sources agree*.

The widget-per-unique-finding-name on the dashboard is built around the same key. Rows are `(asset, finding_name)`; columns are sources. The crosstab fills naturally.

<br>

## Bounty Eligibility as a Tag

A finding inside a public bug-bounty program scope is transactional. A finding outside any program is informational. Most of these findings I can do nothing with — a broken SPF record on a company I'll never touch is trivia. The same record on an asset inside a public bug-bounty program is something I can write up and get paid for. That one tag — in a paid scope, or not — is the only thing standing between me and an evening of scrolling noise.

The resolver runs once per organization at the top of each scan:

```text

disclose.io          → curated policies dataset
hackerone            → public program directory
bugcrowd             → programs.json
security.txt         → /.well-known/security.txt per seed
on-site /security    → scrape for HackerOne / Bugcrowd / Intigriti links
```

Short-circuit on the first hit. About half the roster resolved to either disclose.io or HackerOne; a handful had `security.txt` pointing at HackerOne for a brand the public directories don't list directly. Every downstream finding gets `bounty_eligible: true|false` and `bounty_platform`, and the dashboard's Bounty page filters on the boolean. That page is the one I read tonight; the rest are for debugging when a specific source goes quiet.

<br>

## Landing in Graylog

Findings ship as GELF events over HTTP to a stream named `CTEM Scanner` — same input the other pollers use, no new infrastructure. Custom fields go on the wire with a leading underscore (`_finding_name`, `_severity`, `_sources`, `_bounty_eligible`) and Graylog strips it on the indexing side; widget queries reference the bare name. The `_asset` field flows through the existing client-FQDN lookup table, so a finding on a hostname known to NIOS shows up with its human name in the same column where iLO events do.

The dashboard build is delete-then-recreate-by-title, the same idempotent pattern as the [graylog-deepened](https://www.darknetian.com/posts/2026-05-07-graylog-deepened/) work. Six pages:

| Page | Filter | What It's For |
|---|---|---|
| Overview | none | Finding × source crosstab, totals, top exposure titles |
| Bounty | `bounty_eligible:true` | The working queue |
| Dangling DNS | `kind:dangling_dns` | CNAME takeover candidates |
| Lookalikes | `kind:lookalike_domain` | Brand-impersonation pipeline |
| Admin panels | `kind:admin_panel_exposed` | High-severity public exposures |
| Darknetian | self-monitored | Same shape, applied to my own zone |

{% image "src/assets/img/posts/ctem-spiderweb/dangle.png", "Dangling DNS dashboard page showing 272 findings in 24 hours, a bar chart breakdown by customer with one dominant spike, and a detailed findings table with asset / count / severity / source columns — most rows tagged ctem-scanner or infoblox_ctem at high severity", "prose-img-lg" %}

The Darknetian page exists because the pipeline ran against my own zone first. Same adapters, same dedup, same widgets. If it lies about my own zone — which I can verify by hand — it'll lie about everyone else's.

<br>

## What Surfaced

A first-week snapshot of finding categories, deduped across sources:

| Category | Example | Volume |
|---|---|---|
| DNS hygiene | Missing SPF, broken DMARC, lame delegation | High |
| Subdomain takeover candidates | Dangling CNAME to S3 / Heroku / GitHub Pages | Low, high confidence |
| Exposed admin panels | Jenkins / Grafana / Argo / Vault on public DNS | Rare, highest severity |
| Lookalike domains | Phishing-classified Levenshtein-2 registrations | Variable by brand |
| Brand-impersonation tickets | Phishing kits, credential leaks | Low, vendor already moving |
| Reputation noise | Domains flagged by threat-intel sources | High, mostly filtered |

DNS hygiene dominates. Almost every organization has at least one apex without SPF, with a broken DMARC selector, or with a lame delegation on a subdomain nobody owns anymore. These aren't zero-days, but they enable phishing campaigns downstream and they're the easiest things for an analyst to validate and report. The bounty-eligibility filter rescues them from drowning in reputation noise.

Subdomain takeover candidates are the inverse: rare but immediately actionable. The detector chases each hostname's terminal CNAME, probes the apex over HTTPS, and matches the response body against a fingerprint table — the unmistakable "NoSuchBucket" body from S3, the "There isn't a GitHub Pages site here" string, the Heroku no-such-app page, a dozen more. When one fires, it fires with confidence.

Lookalike registrations with mail records configured are near-certain phishing precursors. Without mail records, they're typosquat-for-resale and lower priority. The pipeline emits both and lets the dashboard sort.

<br>

## What Dogfooding Showed

The darknetian zone produced exactly what I expected: missing SPF on a domain that doesn't send mail, a lame delegation on a `www.` subdomain that's been redirecting for years, no takeover candidates (the homelab is small and quiet), and a small set of typosquat registrations the lookalike service had been accumulating without anyone looking at them. Nothing dramatic. The point wasn't drama; the point was that the dashboard's claims about my zone matched what I knew about my zone. That's the credibility test the rest of the dataset needs to clear, and it does.

The scanner runs at 03:00 and the Bounty page is the first thing I read. Here's the honest part: almost none of my actual roster runs a public program, so most mornings the verdict is "nobody's paying for this one" — which is its own kind of finding. But I've watched the same tag and the same queue turn into real payouts for people pointing it at the right targets. The pipeline doesn't make the money. It just tells you, before coffee, exactly where the money would be.
