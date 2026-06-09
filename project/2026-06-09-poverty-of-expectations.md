---
layout: content.njk
contentType: project
title: "My Cambridge Research Fellowship: What An Intelligence Course Taught Me About Cybersecurity Compliance"
description: In 2023 I spent a month at Cambridge studying intelligence, then wrote a paper removing compliance from security. Three years on, the real problem is a bit more clear.
date: 2026-06-09
tags: [intelligence, compliance, cybersecurity, policy, cambridge]
hero: ./src/assets/img/project/poverty-of-expectations/hero.png
heroAlt: A neon vector illustration in the style of the artillery game Scorched Earth — a tank on a hill firing a shell along a dashed arc that strikes a house on the far hill, over the line "a house takes man-years to build; a shell costs the price of a box of matches"
heroVariant: contain
list: yes
---

In the summer of 2023 I took five weeks off work, flew to England, and lived in a room at Magdalene College while a rotating cast of former intelligence officers taught a small group of us how their trade actually works. I played rugby with the Ely Tigers, I sat in courtyards that inspired CS Lewis and other great minds, and ultimately drank a lot of quality beers. The course was the International Security and Intelligence programme — run by the Cambridge Security Initiative with King's College London's Department of War Studies — and it ended with an exercise that has quietly shaped how I read compliance reports ever since.

<br>

## The Briefing on the Last Day

They handed us the kind of paper a president actually gets. The summer-2001 threat picture: an August 6th President's Daily Brief titled "Bin Ladin Determined To Strike in US," and, underneath it, the field reporting it sat on top of — including an FBI memo out of Phoenix flagging an unusual number of men of interest enrolling in American flight schools. Then they made us be the room. Do you close the airports? Put officers in transit hubs? Pull the students? Say nothing and watch?

{% image "src/assets/img/project/poverty-of-expectations/pdb-aug6-2001.png", "The two-page declassified August 6 2001 President's Daily Brief titled Bin Ladin Determined To Strike in US", "prose-img-lg" %}

<em>The August 6, 2001 PDB, declassified and released April 10, 2004 — see the <a href="https://georgewbush-whitehouse.archives.gov/news/releases/2004/04/text/20040410-5.html">White House archive fact sheet</a> (georgewbush-whitehouse.archives.gov). The flight-school detail came from a separate FBI memo out of Phoenix dated July 10, 2001 — the two were never read against each other.</em>

The lesson wasn't that the analysts were stupid. They weren't. The signals existed. The Phoenix memo existed. The PDB existed. What never happened was the join — nobody positioned to act read the flight-school memo against the strike warning and saw one picture. The course had spent four weeks teaching collection, analysis, and dissemination as a discipline, under Chatham House rules, in seminar rooms small enough that you couldn't hide. The final exercise was the whole loop run live, and the point it drove home was the one Thomas Schelling made in his foreword to Roberta Wohlstetter's <em>Pearl Harbor: Warning and Decision</em>: the failure is rarely a shortage of signals.

> The danger is not that we shall read the signals and indicators with too little skill; the danger is in a poverty of expectations — a routine obsession with a few dangers that may be familiar rather than likely.

<span class="text-neon">A poverty of expectations.</span> I went to Cambridge to study intelligence. I came home unable to read a cybersecurity compliance report without seeing that same poverty staring back out of it.

<br>

## What I Argued in 2023

The research I wrote there asked a question I'd been circling for years in industry: does compliance equate to security? I'd opened the paper's abstract by putting the problem somewhere older than any computer:

> Covetousness, dishonesty, betrayal, thievery, brutality, and other harmful interactions between persons are as old as humanity itself. These problems weren't solved by moving interactions to a digital landscape. Indeed, the opposite trend occurred.

I built the answer out of two things — the published cadence of the major frameworks, and a stack of breach case studies where looking compliant on paper hadn't stopped anything: Colonial Pipeline, Equifax, OPM.

The cadence is the part that's easy to measure, and the numbers are worse than people assume. NIST's Cybersecurity Framework went 1,534 days between version 1.1 and the start of 2.0. ISO 27001 ran 2,938 days between its 2013 and 2022 editions. SOC 2's trust-services criteria sat for 2,787. Average it across the three and you get <mark>2,375 days — about six and a half years between updates of the documents we treat as the definition of "secure."</mark>

Set that against the other curve. The count of <em>known-exploited</em> vulnerabilities — not theoretical CVEs, the ones CISA can prove are being used in the wild — climbs year over year on a line that bends the wrong way. A standard that refreshes every six and a half years is, by construction, describing a threat environment that no longer exists by the time the ink dries. That's the latency thesis, and I still think it's correct.

Latency wasn't where it pointed. I started from three obvious hypotheses: the policymakers are too slow, the organizations are too slow, the attackers out-innovate everyone... the case studies kept landing on the middle one. Companies weren't being breached because NIST was a few years stale. They were being breached because they pursued the certificate in bad faith. I put it bluntly in the paper's own assumptions and limitations:

> A company can be compliant, but not have the processes in place to do things at speed. Showing you can quarantine a device, but then sticking that behind a ServiceNow ticket that goes to someone out of office means the breach happens.

The firewall had its explicit-deny rule for the auditor, but nobody checked the thousand rules above it. The CISO reported to a CIO who reported to a COO, three steps removed from anyone accountable to a customer. There was no punishment for failure and no legal imperative to comply, so the rational move was to gamble that it-won't-be-us. I concluded the organizations were to blame.

Importantly, I chose to differentiate this from victim blaming. Policy is a beautiful problem space: as a standard do you pick language that it so difficult to achieve (and therefore secure) that is carries weight? Or, do you live somewhere that is easier to attain at the cost of gamified systems that hopefully broadly move the needle or discussion? 

<br>

## Why That Conclusion Was the Wrong Shape

Three years and a lot of customer conversations later, I think "who's to blame" was the wrong question, and I'd built the whole paper around it.

Blame assumes someone in the system is failing at a job they could be doing well. But every actor I named was behaving rationally inside the incentives they actually face. The standards bodies move slowly because consensus is slow and the cost of a bad revision is high. The organizations buy the cheapest defensible posture because nobody bills them for the gap between the certificate and the truth. None of them are culprits to be identified as the main villain. <span class="text-neon">The structure is.</span>

Schelling, again, said it first (this time in his "Arms and Influence") — and this time it's the line that opens the same foreword, before he ever gets to expectations:

> One of the lamentable principles of human productivity is that it is easier to destroy than to create. A house that takes several man-years to build can be burned in an hour by any young delinquent who has the price of a box of matches. Poisoning dogs is cheaper than raising them. And a country can destroy more with twenty billion dollars of nuclear armament than it can create with twenty billion dollars of foreign investment.

That is the whole asymmetry, and it predates every framework I was measuring. Defense has to be comprehensive and continuous and correct on a six-and-a-half-year refresh cycle. Offense has to be cheap and occasional and right once. No update cadence closes a gap like that, because the gap isn't a latency bug in the standards — it's the <span class="text-neon">cost curve of destruction versus creation</span>, which compliance was never built to bend.

<div class="terminal-card p-3 my-4"><strong class="h-mono">THE MODERN BOX OF MATCHES</strong><br><br>Schelling's delinquent needed the price of a box of matches. The modern equivalent is that we wouldn't be having most of this conversation if cryptocurrency hadn't trivialized the movement of dark money to the people doing the destroying. Cybercrime is a business model because there is finally a frictionless rail to get paid on it. The match got cheaper, and the incentive asymmetry got a payment processor. This feedback loop will accelerate as the time-to-task diminishes through adoption of AI.</div>

<br>

## What Cambridge Actually Changed

The thing I carried out of that month wasn't a better framework. It was a different job description for the work.

The intelligence cycle I'd been taught — collect, analyze, disseminate, and act before the adversary does — is the same loop a threat-intelligence team runs every day, and it is the loop compliance quietly opts out of. An audit is a snapshot of yesterday's controls graded against a standard from years before that. It is, almost precisely, Schelling's poverty of expectations rendered as a checklist: a routine obsession with the dangers that are familiar rather than the ones that are likely. The Phoenix memo and the August 6th brief never got joined because nobody's expectations had room for the join. Most compliance programs have the same blind spot, institutionalized and signed off on annually.

So the recommendation I'd give now is narrower and more demanding than the one in the paper. Stop treating compliance as the security program and start treating it as <span class="text-neon">the floor</span>. Run the intelligence loop on top of it — adversary-facing, continuous, accountable to someone who answers to a customer — because the certificate will always describe a world that has already moved, and the people trying to burn the house down are working off a much shorter cycle than your auditor.

I don't think the frameworks are useless. I think we mistook the warning light for the engine. The signals were never the problem. <span class="text-neon">Our expectations were.</span>

<br>

<div class="terminal-card p-3 my-4"><strong class="h-mono">PROVENANCE</strong><br><br>This essay is the prose version of a research document I wrote for the 2023 International Security and Intelligence programme. The required submission was capped at 3,500 words, which was never enough room for the argument; the fuller ~70 page document — case studies, the CVE regression, the framework-latency tables, and the excerpts quoted above — is on GitHub: <a href="https://github.com/nicknacnic/research/blob/main/ISI_Research_Proposal.pdf">ISI_Research_Proposal.pdf</a>. Specifics on the August 2001 documents are drawn from the declassified PDB and the 9/11 Commission record, not from memory.</div>
