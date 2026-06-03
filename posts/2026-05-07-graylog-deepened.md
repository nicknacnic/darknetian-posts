---
layout: content.njk
contentType: blog
title: Graylog Enrichment, Deepened
description: Adding MAC→DHCP-hostname lookups, dashboards-as-code, and the long tail of NIOS WAPI and OpenSearch quirks the first pass left behind.
date: 2026-05-07
tags: [graylog, infoblox, siem, homelab]
hero: ./src/assets/img/posts/graylog-deepened/hero.png
heroAlt: Top clients widget showing client_ip resolved next to fixedaddress name and DHCP hostname columns, with NIOS-known hosts alongside unenriched ones
list: yes
---

The [previous enrichment post](https://www.darknetian.com/posts/2026-01-08-enriching-graylog/) wired NIOS PTR records into Graylog so `client_ip` could become `client_fqdn` everywhere. That gets you 80% of the way — but the 20% that's left is the noisy half. PTR records exist for the things you statically named (switches, hypervisors, the printer that should not exist). They do *not* exist for the iPhone that joined the guest SSID this morning. Those clients live in DHCP, keyed by MAC.

This post is the sequel: how I closed that gap, plus the quirks that turned up while moving the whole config into a `graylog` repo under git so a clean Graylog can be rebuilt with one command.

<br>

## The MAC gap

Cradlepoint and Aruba both log "new client" events with the MAC address, not the IP — Aruba reports `aruba_client_mac`, Cradlepoint reports `cp_new_client_mac`. PTR enrichment can't help. To resolve `b8:27:eb:42:11:0a` into `living-room-pi`, you need NIOS to tell you it handed that MAC a lease.

NIOS exposes two relevant objects: `fixedaddress` (static DHCP reservations — what you actually defined) and `lease` (the running state, including which dynamic clients are currently bound). Merging both gives the broadest coverage: a fixedaddress.name wins when set, lease.client_hostname is the fallback.

The exporter is a small Python script that pulls both, filters down to the homelab subnet, and writes a `mac,hostname` CSV that a Graylog file-backed data adapter serves to the pipeline. A systemd timer runs it hourly.

<br>

## WAPI quirks

This is where it got interesting. A few hours that the docs didn't save me.

**`record:fixedaddress` doesn't exist.** The DNS-record family — `record:a`, `record:ptr`, `record:cname`, `record:host` — all take the `record:` prefix. DHCP objects do not. `fixedaddress` is the bare object name; prefixing it gets you `Unknown object type`. Same for `lease` and `network`.

**`lease.binding_state` is not server-side searchable** on WAPI v2.13. You'd think filtering for `binding_state=ACTIVE` would let the grid send you only the live leases — it does not. The grid returns `Field is not searchable`. So you pull every lease and filter client-side:

```python

leases = wapi_list(sess, base, "lease",
                   return_fields="hardware,client_hostname,address,binding_state")
leases = [ls for ls in leases if (ls.get("binding_state") or "").upper() == "ACTIVE"]
```

**Paging requires both `_paging=1` *and* `_return_as_object=1`.** Set one without the other and the second page comes back malformed — the response shape changes between pages and your parser will silently truncate at the first 5000 records. Always set them as a pair:

```python

params = {
    "_return_fields": "mac,ipv4addr,name",
    "_paging": "1",
    "_return_as_object": "1",
    "_max_results": "5000",
}
```

<br>

## Normalizing the MAC

With the CSV in place and the `infoblox-nios-mac` lookup table wired up, the pipeline does three things:

```text

rule "mac normalize cp_new_client_mac"
when has_field("cp_new_client_mac") && !has_field("client_mac")
then set_field("client_mac", lowercase(to_string($message.cp_new_client_mac)));
end

rule "mac normalize aruba_client_mac"
when has_field("aruba_client_mac") && !has_field("client_mac")
then set_field("client_mac", lowercase(to_string($message.aruba_client_mac)));
end

rule "mac enrich client_mac"
when has_field("client_mac")
then
  let mac = to_string($message.client_mac);
  let host = lookup_value("infoblox-nios-mac", mac);
  set_field("client_hostname", to_string(host));
end
```

Source-specific fields get coalesced into one `client_mac`, the lookup writes `client_hostname`. Now any widget keyed on the unified fields displays a human name regardless of which AP or router generated the log.

<br>

<div class="terminal-card p-3 my-4"><strong class="h-mono">Display precedence</strong><br>When a widget has to pick one identifier to show, the rule across the dashboards is: client_hostname (DHCP) → client_fqdn (PTR) → client_mac → client_ip. DHCP names beat PTR names because they reflect what the device announced about itself, not what someone typed into IPAM in 2019.</div>

<br>

## Dashboards as code

The other thing this round was about: getting every pipeline rule, lookup table, index set, stream, input, and dashboard out of the Graylog UI and into a git repo that can rebuild a stock Graylog in one command. The result is [`apply_all.py`](https://github.com/nicknacnic/graylog), a thin orchestrator over five idempotent stages:

```text

1. index sets   — created first so streams can be repointed
2. streams      — created or repointed to the new index sets
3. inputs       — GELF HTTP for the iLO poller, raw UDP for Aruba syslog
4. lookups      — infoblox-nios-mac adapter + cache + table
5. pipelines    — Cradlepoint, Aruba, VMware, MAC Enrichment,
                  plus a destination_fqdn splice into the existing
                  Enrichment pipeline you already have running
6. dashboards   — last, because they reference everything above
```

Re-running on a healthy instance is a no-op for everything except dashboards, which are delete-then-recreate-by-title — the only way to keep widget layouts under code control until Graylog ships a real `PUT /views`.

<br>

## Pivots, IDs, and tables that won't render

The first dashboard I built from code rendered every row label correctly and every value as blank. Empty table cells, with the right row count and the right column names.

The fix took longer than it should have: in Graylog's table renderer, a pivot's `id` and the parent widget's `config.name` have to match exactly. Two unrelated fields, one identical string. If they drift apart, the renderer can't connect the column to the values, so it draws the structure with no data. There is nothing in the API response that says this; the search results look fine, the widget config looks fine. You stare at JSON for an hour.

The lib helper that fixes it is one line of business logic and saves every future dashboard:

```python

def align_pivot_ids(widget: dict) -> dict:
    name = widget["config"]["name"]
    for pivot in widget["config"].get("row_pivots", []):
        pivot["id"] = name
    for pivot in widget["config"].get("column_pivots", []):
        pivot["id"] = name
    return widget
```

Run it on every widget before posting the view. Trivial. Took *forever* to find.

<br>

## The 1000-field wall

VMware's vCenter+ESXi firehose is ~2.2M messages/day. After a few weeks of running, a chunk of those started failing to index. The Graylog "Indexer Failures" page showed 188K errors on `graylog_12` with the same shape:

```text

Limit of total fields [1000] in index [graylog_12] has been exceeded
```

OpenSearch caps a single index's mapping at 1000 fields by default. ESXi messages include a long tail of nested `vmware_app.*` properties; combined with everything else flowing into the default index set, the mapping pushed past the cap and new fields stopped being added (which silently dropped messages).

Two options: raise the cap, or partition. I went with partition — every high-cardinality source now gets its own index set:

```text

iLO Redfish      → ilo_redfish_*
VMware           → vmware_*
Palo Alto        → panos_*
Infoblox UDDI    → uddi_*
Infoblox NIOS    → nios_*
Aruba AP         → aruba_*
```

Each script in [`indexing/`](https://github.com/nicknacnic/graylog/tree/main/indexing) creates the index set with `TimeBasedSizeOptimizingStrategy` rotation and atomically repoints the existing stream(s) from the default set. The default index gets a fresh rotation afterward so the new clean index doesn't inherit the bloated mapping. The pattern lives in `indexing/ilo_redfish.py` if you want to copy it.

<br>

{% image "src/assets/img/posts/graylog-deepened/vmware-inventory.png", "VMware Inventory dashboard page showing 2.2M messages, 76 apps, sources pie chart, and a Top 30 apps list — all flowing into the dedicated vmware_* index set", "prose-img-md" %}

<br>

The VMware Inventory page above is what falls out of the partition: every `vmware_app.*` field has somewhere clean to live, the firehose stops eating the default index's field budget, and the "Top 30 apps" pivot can finally render without the half-mapped fields it used to fight with.

<br>

## Splicing one rule into someone else's pipeline

The pre-existing `Enrichment` pipeline already handled `client_ip` → `client_fqdn`, `sender_ip` → `sender_fqdn`, and the MaxMind geo lookup, all in stage 2. I wanted to add `destination_ip` → `destination_fqdn` (so the Palo Alto Top Destinations table could show real names instead of just IPs) without rewriting the whole pipeline definition or risking the user's existing rules.

The splicer fetches the live pipeline source, parses the stage 2 body, inserts a `rule "ipam enrich destination_fqdn";` line if it's not already there, and PUTs the result back. Idempotent: re-running is a no-op when the rule is present. The new rule itself is the same shape as the existing ones, just keyed on `destination_ip`:

```text

rule "ipam enrich destination_fqdn"
when has_field("destination_ip")
then
  let ip = to_string($message.destination_ip);
  let fqdn = lookup_value("infoblox-nios-ptr", ip);
  set_field("destination_fqdn", to_string(fqdn));
end
```

Top Destinations now shows `nas.darknetian.com` for the internal traffic and `unknown` for everything external (which is honest — the LAN PTR set doesn't cover the public internet).

<br>

## What's in the repo

Everything mentioned here is in [github.com/nicknacnic/graylog](https://github.com/nicknacnic/graylog):

- `apply_all.py` — the one-shot orchestrator
- `indexing/` — per-source index sets + stream repoint scripts
- `pipelines/` — JSON specs for Cradlepoint, Aruba, VMware, MAC Enrichment + the `destination_fqdn` splicer
- `lookups/mac_to_hostname.py` — Graylog data adapter / cache / lookup table creation
- `tools/nios_mac_to_graylog_csv.py` — the NIOS MAC exporter
- `dashboards/` — multi-page view builders (iLO, VMware, Palo Alto, Infoblox Ops) plus single-page ones (Cradlepoint, Aruba)
- `lib/graylog.py` — stdlib HTTP client + widget/search/view builders, including the `align_pivot_ids` helper that earns its keep

Next on the bench is a threat-intel lookup chain — pulling Infoblox TIDE indicators into a Graylog adapter so Palo Alto and UDDI streams can tag matches at ingest. That's the third post in this enrichment arc. Until then, happy hunting.
