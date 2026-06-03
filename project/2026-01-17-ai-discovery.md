---
layout: content.njk
contentType: project
title: Agentic AI Discovery
description: The fight for an open web continues.
date: 2026-01-17
tags: [ai research, hot rfc, dns, dns-aid, ops]
hero: ./src/assets/img/project/ai-discovery/hero.png
heroAlt: A meme from the 'you wouldn't download a car' ad campaign that says 'you wouldn't add to DNS'
list: yes
---
# Overview
The pace of innovation regarding how AI agents SHOULD connect and exchange application data with one another is constantly evolving. Each day, there's a new and exciting proposal outlining the future of what the internet might provide. 

<br>

## Problem Statements 
There's no shortage of participation, but prior to the IETF creating a working group to publish RFCs standardizing web communications, there is typically a Birds of a Feather which outlines problem statements. It's tough to solve issues without mutual agreement on the nature and boundaries of it to begin with. 

There are a few sources for drafts, but many Standards Development Organizations (SDOs) are private or charge a membership, so I will not be discussing those. I will use the IETF drafts as starting points. 
- [Internet of Agents problem statement](https://datatracker.ietf.org/doc/draft-ye-problems-and-requirements-of-dns-for-ioa/00/)
- [AI Agent Discovery problem statement](https://datatracker.ietf.org/doc/draft-mozley-aidiscovery/)
- [HTTP Agent Discovery](https://datatracker.ietf.org/doc/draft-cui-ai-agent-discovery-invocation/)
- [DMSC Architecture](https://datatracker.ietf.org/doc/draft-li-dmsc-architecture/)

Yes, the second draft is partially my contribution, which aligns to the free and open internet the IETF promotes. 

<br>

## Internet Drafts
Once problem statements are created, charters of working groups are drafted, and then working groups collaborate on modifying internet drafts to publish RFCs. There are a few drafts relevant in this problem space:
- [Agent Name Service](https://www.ietf.org/archive/id/draft-narajala-ans-00.html)
- [BANDAID](https://www.ietf.org/archive/id/draft-mozley-aidiscovery-00.txt)
- [CATS](https://datatracker.ietf.org/doc/draft-liu-cats-dns-service-discovery/00/)
- [Agent Networks Framework](https://datatracker.ietf.org/doc/draft-zyyhl-agent-networks-framework/)
- [Agent Naming Resolution](https://datatracker.ietf.org/doc/draft-cui-dns-native-agent-naming-resolution/)
- [Agent Discovery](https://datatracker.ietf.org/doc/draft-pioli-agent-discovery/)


I'd be remiss to not mention the nearly unending amount of open-source, and closed-source work in the space as well, some examples:
- [A2A Gateway](https://github.com/Tangle-Two/a2a-gateway?tab=readme-ov-file)
- [Agent Community](https://github.com/agentcommunity/agent-interface-discovery)
- [Google UCP](https://developers.google.com/merchant/ucp)
- [GoDaddy ANS](https://www.godaddy.com/en-ph/ans)

### Analysis
When I read these problem statements, I'm reminded of a few core tenants of the internet which seem to be overlooked. Many times the approaches risk fragmentation, centralization, and loss of governance (proprietary / opaque registries or needless centralization). What incentive exists for a registry operator to allow migration off their platform? Or allow interoperability with another?

Some of these proposals relegate the beautiful internet to the role of commercial shipping provider: advertising your networks, available data, and costs to transit and interact. 

Worse yet, how might the market dis-incentivize an operator from engaging in anti-competitive and monopolistic behavior, restricting access to the internet? These are problems that require novel solutions.

There is not yet enough discussion to create consensus on the matter through a number of SDOs. Some folks truly believe discovery lives in the application layer, others think it's better as its own service discovery layer. 