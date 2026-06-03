---
layout: content.njk
contentType: blog
title: Encrypted DNS Server Redirection
description: Discussing the merits of IETF internet draft EDSR. 
date: 2026-01-10
tags: [hot rfc, ops]
hero: ./src/assets/img/posts/hot-rfc/edsr/hero.png
heroAlt: Secure DNS configuration menu in Brave browser settings
list: yes
---
If you're unfamiliar with the IETF process, here's a high level overview:
- Any author is welcome to submit an internet draft (policy proposal that doesn't have a 'home')
- These drafts are reviewed by the 'dispatch' team, which route to various 'areas' within the IETF, for example, the internet area
- The internet area dispatch team then decides if this work is addressable by an existing 'working group', the folks with shared experience and charter to output related RFCs
- If the WG exists, it is then worked on up until a time it may be published. This process is typically a timeline of years
- If the WG doesn't exist, the timeline is longer until charters are made and stakeholders are gathered
___
You may read the relevant draft [here](https://datatracker.ietf.org/doc/draft-ietf-add-encrypted-dns-server-redirection/). My read is the debate stems from should DNS policy be a matter of BGP implementation (is this the best place to express application-layer traffic)? Until the internet area working group decides on where this policy expression occurs, we will see related drafts. 

EDSR is a proposal stemming from NOV 2022, and is picking up steam in the encrypted DNS world. Many open DNS providers (cloudflare, quad9, google, etc) all provide DOT/H/Q capabilities. Browsers and various clients supporting designated DNS resolver discovery (DDR) are also able to opportunistically upgrade from cleartext to encrypted communication channels assuming the DNS server supports opportunistic DOT/H/Q.

In short, DOT/H/Q is generally regarded as a good thing for anti-censorship and promoting consumer privacy. In my experience, enterprise security experts don't reflect fondly on the technology as it imperils network traffic visibility. 

The 'meat' of the proposal centers around one idea: have the recursive resolver tell the client which resolver is preferred. Or rather, create a mechanism in which a client 'asks again' after upgrading to encrypted communications which server is best to use within a domain. The draft specifies a new method (not protocol-dependent) for this mechanism. 

The recursive resolver has the option to pass the session to another server. This as escaping the 'tyranny' of anycast, which is deterministic via BGP. This draft allows anycast to be the beachhead IP, but then allows for a 'better' server to be provided (operators steer based on upcoming maintenance, for example).

EDSR replaces the method by which the client receives the list of resolver upstreams. It *doesn't* tell a client how to behave between them or remove DHCP from specifying which server to use. These are all OS and application-dependent, the scope of the draft is operator-targeted. 

For the enterprise: an organization gives out a single IP or two, allow DOT/H/Q upgrade to get a recursive resolver to point them somewhere locally. This provides a mechanism to stack your hardware (big site, smaller sites elsewhere). Once an edge DNS server is configured in a network, you add complexity. DHCP changes, if there's an outage single point of failure etc.. This is a mechanism to, from an upstream source, create policy from where client traffic goes towards the apex.

There's existing scholarship and development work in this area. See DNS-OARC, for example [a recent talk by Geoff Houston](https://indico.dns-oarc.net/event/52/contributions/1137/attachments/1088/2226/2025-02-10-oarc44-recursives.pdf) which captures an undesirable behavior of DNS servers: the selection and sorting algorithm *frequently selects suboptimal roots*. 
{% image "src/assets/img/posts/hot-rfc/edsr/expectation.png", "A screenshot of the expected DNS server attachment preference", "prose-img-md" %}
{% image "src/assets/img/posts/hot-rfc/edsr/reality.png", "A screenshot of the DNS reality server attachment preference for BIND 9", "prose-img-md" %}
{% image "src/assets/img/posts/hot-rfc/edsr/server.png", "A screenshot of the DNS server attachment preference", "prose-img-md" %}

This isn't the only scholarship, but I use it as an example to prove a point: even with BGP the expectation of closest DNS server diverges from reality. For the most time-extensive operation, a recursive resolution to root not from cache, it takes an additional 2X overhead due to the selection algorithm. I assume this challenge exists with opportunistic upgrading as well. 

Today, if your client latches onto the wrong DNS server for encrypted resolution you could experience shortcomings in performance as well as jurisdictional issues with regards to data treatment (i.e. GDPR).

All this to say, thanks to the coauthors for a great suggestion. As it solves this, and a few other, operations-focused problems. Let's see how many recursive DNS server operators take up the charge.