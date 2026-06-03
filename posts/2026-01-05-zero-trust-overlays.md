---
layout: content.njk
contentType: blog
title: Zero Trust Overlays
description: Comparison of Zero Trust documents from a US policy perspective.
date: 2026-01-05
tags: [compliance]
hero: ./src/assets/img/posts/zto/hero.png
heroAlt: Stock market ticker
list: yes
---
<span class="text-warn">Originally posted on Infoblox Corporate Blog on 17 APR 2025 as "LEVERAGING DNS TO ACCELERATE ZERO TRUST ADOPTION: PART 1" ([link](https://www.infoblox.com/blog/security/leveraging-dns-to-accelerate-zero-trust-adoption-part-1/))</span>

In June 2024, the U.S. Department of Defense (DOD) released likely the most consequential document on Zero Trust. The Zero Trust Overlays (ZTO) [1] is comprehensive in scope and provides maturity levels for organizations as they migrate from perimeter-based defense. The DOD anticipates their Zero Trust project to span over five years. What does the ZTO mean for commercial enterprises? How is DNS a critical component to Zero Trust adoption? 

- *“Zero Trust is much more than an IT solution. Zero Trust may include certain products but is not a capability or device that may be bought.” —John Sherman, DOD CIO* 

Organizations with mature cybersecurity posture know adopted compliance frameworks (NIST, SOC2, ISO, etc.) don’t necessarily equate to effective delivery of cybersecurity. Compliance frameworks focus on information management and typically not on risk mitigation. The difference is key: compliance frameworks mitigate risk as a byproduct of adoption, whereas the Zero Trust methodology explicitly focuses on mitigation and the byproduct is efficacy. 

Zero Trust is difficult for organizations to adopt because the onus in operationalizing technology is frequently offloaded onto defenders. Organizations suffer double in lieu of paying for technology; they are then required to acquire/build the requisite expertise for fine-tuning prior to realizing security value. The outcomes are further imperiled as they’re predicated upon recruiting, retaining and training effective operators to recall, execute and plan according to the department charter. Succession planning is a stretch goal for even the most mature organizations. 

The ZTO aims to be a blueprint for organizations to understand the context behind a control, to better apply technology to business process, which is sometimes lost in a checkbox of a baseline. Pertinent example from SOC2: 

- CC6.6.2: Every production host is protected by a firewall with a deny-by-default rule. Deny-by-default ruleset is a default on the Entity’s cloud provider. 

In the above excerpt of a SOC2 Type 1 report the control requires a rule demonstrating positive control on firewalls, but it’s not nearly specific enough. What about non-standard ports? Decryption? These, and more, potentially compromise a firewall’s capability to effectively deliver security. The difference is like night and day in the ZTO: 

- SC-7(17): Enforce adherence to protocol formats. System components that enforce protocol formats include deep packet inspection firewalls and XML gateways. The components verify adherence to protocol formats and specifications at the application layer and identify vulnerabilities that cannot be detected by devices operating at the network or transport layers [SC-7(17)]. [2] 

The ZTO is ensuring applications flow with the correct port, protocol and decrypt actions as an enabler to a key pillar in addition to only allowing rules for known acceptable traffic flows. However, problems compound from here. Organizations have an unenviable choice to partially enable the Data Loss Prevention (DLP) pillar: micro-segment networks with additional enforcement points, or hairpin all network traffic back to the current enforcement point(s) to meet the stated requirement: 

- AC-4(2): Use protected processing domains to enforce flow control policies as a basis for flow control decisions 	[AC-4(2)]. [3] 

This choice, and many like it, prevents organizations from properly adopting Zero Trust. Creating more perimeters with micro-segmentation is not a first choice for most due to scaling costs paired with architecture complexity. Hair-pinning traffic creates a trade-off on performance for colleagues, applications and critically—customers. 

The above deliberations ensure most organizations opt to not spend the additional money or incur additional operational overhead. How do we escape this paradigm?

In theory, Zero Trust is easy. In practice, it’s hard. Companies need an accurate inventory of assets, strong documentation on change controls and policies, and then between the two of those, also vast high-fidelity telemetry with analysis to equip defenders to make the best decisions possible. 

As a result, many organizations look to DNS as a key enabler to remove the barriers to Zero Trust adoption. Why? It’s every device’s intent to communicate with outside networks. DNS is the bridge: it benefits from its architecture being inline like an NGFW, while also possessing institutional strength from correlation, like XDR. Infoblox is uniquely positioned to provide comprehensive insights of assets as the only vendor seamlessly integrating public and private cloud DDI data in one console. 

Using DNS to profile devices, build an inventory and as a telemetry source revolutionized Zero Trust progress for many Infoblox clients this past year. A month prior to the publication of the ZTO, Microsoft announced their vision of Zero Trust DNS in collaboration with the DOD, that is, forcing clients to query a trusted DNS server prior to the host firewall ports being opened and traffic flowing. This is a great start to secure the first mile of DNS between a client and recursive resolver, but is weak without encryption on the last mile between the resolver and root servers provided from a solution like Infoblox Threat Defense™. Pairing the two are required for compliance with EO 14144 [4]. In effect, the world is looking to DNS as a first line of identification of defense, and Infoblox is a critical component in doing so. 

Vendors, commercial organizations, compliance policymakers and all cybersecurity stakeholders are waking up to the defensive capabilities afforded by Protective DNS and the knock-on effects for speeding Zero Trust adoption. 

The way we do this today is through our Protective DNS solution, Threat Defense, and DDI analysis engine, Infoblox Universal Asset Insights™. The latter solution is correlation of multiple sources: DHCP fingerprints as a device joins a network, cloud correlation of devices based off device telemetry, combined with data from authoritative DNS. For example, what happens when I have an online host but no corresponding PTR or A record? What happens when I have an A and PTR record but no online host? Identifying zombie and dangling assets are but a small part of the data we use to create better compliance outcomes. 

In the next installment of this series, we will review these specific data types, compliance outcomes regarding Zero Trust adoption, provided by [Threat Defense](https://www.infoblox.com/products/threat-defense/) and [Universal Asset Insights](https://www.infoblox.com/products/universal-asset-insights/). Stay tuned! 

Infoblox Resources: 
[NCSC Protective DNS](https://www.infoblox.com/blog/security/the-u-k-s-ncsc-recommends-protective-dns-for-government-and-industry/)

Footnotes
1. https://web.archive.org/web/20250306014912/https://dodcio.defense.gov/Portals/0/Documents/Library/ZeroTrustOverlays.pdf
2. ZTO, page 226
3. ZTO, page 314
4. https://www.federalregister.gov/documents/2025/01/17/2025-01470/strengthening-and-promoting-innovation-in-the-nations-cybersecurity