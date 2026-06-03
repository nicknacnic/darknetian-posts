---
layout: content.njk
contentType: project
title: Creating a New DNS Resource Record Type
description: Why it may be beneficial to think about DNS in a new way.
date: 2026-01-18
tags: [dns, dns-aid, ai research, ops]
hero: ./src/assets/img/posts/new-rr/hero.png
heroAlt: A meme from the 'you wouldn't download a car' ad campaign that says 'you wouldn't add to DNS'
list: yes
---
# Introduction
DNS, at its core, is used by humans for ease of remembering the location of a web service. As opposed to memorizing an IP address of the site you wish to visit, you remember 'example.com', type example.com in your browser, and through DNS your browser takes you to the IP address where example.com is hosted. 

At the time, a most welcome and wonderful innovation. However, this medium leaves humans vulnerable to typos, phishing, and other attacks to subvert their purported trust in a site. In the era of AI agents, it's preferable to embed a higher-density data medium to include discovery metadata. Agents need not rely on semantics, but the metadata output (cost, location, modalities) may influence which agent the requesting agent selects. 

When Tim Berners Lee invented the World Wide Web, he envisioned a world where all human minds would be interconnected, sharing emotions and ideas at the speed of thought, like a room we are all able to converse in at the same time. He didn't yet know he was right, but not for humans (yet), but for agents. 

In the future, agents may work together to accomplish tasks. These agents may eventually function like neurons in the brain, and need to find one another with higher-fidelity data sources, like vectors. Imagine agent discovery completing through cached answers on DNS servers 'knowing' where a nearest neighbor may possess the needed skillset for the job.

<br>

## Philosophy
A full JSON “agent card” describing model, training, certs, policies, toxicity, cost, location will be kilobytes to megabytes. DNS simply won’t carry that raw, even with EDNS0, without running into fragmentation and operational pain. In practice DNS could carry several projections of the same underlying object:
- A cryptographic commitment to the full metadata
- A compressed semantic summary that’s machine-usable without an HTTP round-trip
- Optional pointers for fetching the full, rich document

The semantic summary might be described as a "canonical descriptor hash" where:
- Deterministic ordering of keys
- No insignificant whitespace
- Deterministic encoding of numbers/booleans/nulls
- Compute AgentID = hash(canonical_descriptor) using e.g. SHA-256 or BLAKE3.
    - This 32-byte value is your primary dense identifier. It’s tiny, stable, and can commit to arbitrarily large data.
- Capability / policy summary bits

These suggestions give a pathway to pull a very small set of critical properties directly into DNS:
- Safety flags: supports_toxicity, pii_stripping, jailbreak_hardening_level
- Modality bits: chat, vision, speech, code, etc.
- Jurisdiction / data-residency bits: EU-only, US-only, etc.
- Represent as bitfields and small enums rather than text. This lets your DNS packet answer first-pass policy questions without leaving DNS at all.
- Semantic vector / embedding

Then an agent would treat the full descriptor as a document and compute an embedding in some N‑dimensional space. Example: 
- 96‑dim vector with 8-bit signed ints → 96 bytes.
- Store that embedding directly in RDATA as a binary vector.

This gives client software a way to:
- Do nearest-neighbor search among agents it has already cached
- Score “distance from my desired capability profile” using just DNS answers
- Pointers + integrity
- Store at least one URI to the full descriptor (https://example.com/.well-known/agent.json) and a hash of that document.

This “hash + locator” pattern is already used in DNS-based AI discovery work: rich metadata lives at HTTP, with a hash stored in DNS so clients can verify integrity. A single record can conceptually contain:
- A canonical AgentID hash
- A tiny capability/policy bitfield
- An optional embedding vector
- One or more URIs + hashes for full metadata / attestations

<br>

## The Problem
I've written briefly on the general problem space with AI agents [here](https://www.darknetian.com/project/2026-01-17-ai-discovery/), the skinny is that there's not much consensus. Application developers wish to embed application-specific metadata in JSON blobs called 'model cards' highly reminiscent of machine learning model cards. 

However, AI agents that are interacted with by humans in a chat interface may need the ability to walk on both sides: the agentic web, the human web. How might we accomplish this?

<br>

## The Solution
I choose a DNS-centric approach. It's my weapon of choice: there is international federated governance providing global uniqueness, it's decentralized, and individual operators preserve their sovereignty through its use compared to centralized registries. 

The IETF DNSOP WG wouldn't likely add a new RR-type, and certainly not from my bad code (lol). However, I want to learn, experiment, and prove it is possible. 

<br>

## The Environment
To facilitate learning, let's work with CoreDNS. I'm familiar enough with ISC's BIND, but what of CoreDNS? This brief writeup assumes you've already installed git, go, and docker.
```bash

g clone https://github.com/coredns/coredns
cd coredns
g checkout b $name # where $name is your branch
g init # in the event you want to push changes to your repo
make # build the binary locally
```

From here we'll make the docker image:
```bash

sudo docker run --rm -it \
  -v "$PWD":/go/src/github.com/coredns/coredns \
  -w /go/src/github.com/coredns/coredns \
  golang:1.24 sh -c 'GOFLAGS="-buildvcs=false" make gen && GOFLAGS="-buildvcs=false" make'
```

Tell docker to use our local files:
```bash

docker build -t coredns-aid:local .
```

Create an example zone file:
```zone

cat > Corefile <<'EOF'
example.com:1053 {
    file /zones/example.com.zone example.com
    log
    errors
}
EOF

mkdir -p zones

cat > zones/example.com.zone <<'EOF'
$ORIGIN example.com.
$TTL 300
@   IN SOA ns1.example.com. hostmaster.example.com. (
        1       ; serial
        3600    ; refresh
        600     ; retry
        604800  ; expire
        86400   ; minimum
)
    IN NS  ns1.example.com.
ns1 IN A   127.0.0.1
EOF
```

Reload the docker instance and run your first query against it (i.e.```dig @127.0.0.1 -p 1053 ns1.example.com A +noall +answer```)

{% image "src/assets/img/posts/new-rr/dig.png", "A screenshot of the dig output", "prose-img-md" %}

<br>

# Data 

<br>

## Model Cards
I choose to use OpenAI's ChatGPT retrieval plugin [here](https://github.com/openai/chatgpt-retrieval-plugin/blob/main/.well-known/ai-plugin.json). Yes, this data isn't AI-workload specific, yet in the future we may see classifications of many types of metadata: training set, location, cost, modalities, certifications, toxicity support, etc.
```json

{
  "schema_version": "v1",
  "name_for_model": "retrieval",
  "name_for_human": "Retrieval Plugin",
  "description_for_model": "Plugin for searching through the user's documents (such as files, emails, and more) to find answers to questions and retrieve relevant information. Use it whenever a user asks something that might be found in their personal information.",
  "description_for_human": "Search through your documents.",
  "auth": {
    "type": "user_http",
    "authorization_type": "bearer"
  },
  "api": {
    "type": "openapi",
    "url": "https://your-app-url.com/.well-known/openapi.yaml",
    "has_user_authentication": false
  },
  "logo_url": "https://your-app-url.com/.well-known/logo.png",
  "contact_email": "hello@contact.com", 
  "legal_info_url": "http://example.com/legal-info"
}
```

<br>

## DNS Resource Records

<br>

### Minimally Invasive
Using CoreDNS, you're able to specify undefined resource records. You'd take the data, hash it into a compact commitment, and publish that commitment as a generic/unknown RR using the [RFC 3597](https://www.rfc-editor.org/rfc/rfc3597)  ```\# <len> <hex>``` presentation format.

So let's pick a minimally-viable implementation of an embedded model card with the following data:

- version (1 byte) = 0x01
- flags (2 bytes) = 0x0000
- hash_algo (1 byte) = 0x01 meaning SHA-256 (our convention)
- agentid_len (1 byte) = 0x20 (32)
- agentid (32 bytes) = SHA-256 of canonical JSON (sorted keys, no whitespace)
- uri_count (1 byte) = 0x01
- uri_hash_algo (1 byte) = 0x01 (SHA-256)
- uri_hash_len (1 byte) = 0x20
- uri_hash (32 bytes) = same hash (commits URI target content by canonical form)
- uri_len (1 byte) = length of URI
- uri (N bytes) = ASCII URI pointing to the raw GitHub JSON, HTTP well-known, etc.

Example zone file containing this data:
```zone

$ORIGIN example.com.
$TTL 300
@   IN SOA ns1.example.com. hostmaster.example.com. (
        1       ; serial
        3600    ; refresh
        600     ; retry
        604800  ; expire
        86400   ; minimum
)
    IN NS  ns1.example.com.
ns1 IN A   127.0.0.1

; Experimental AI Descriptor (AID) as generic RFC3597 record
; TYPE65400 RDATA = v1 + sha256(canonical ai-plugin.json) + pointer URI
agent1._aid  IN  TYPE65400  \# 170 (
  0100000120354edc0a0f26dd973dd6c6
  81338bb189bb42137697484d795b4b5d
  7d0426c1fb010120354edc0a0f26dd97
  3dd6c681338bb189bb42137697484d79
  5b4b5d7d0426c1fb6168747470733a2f
  2f7261772e6769746875627573657263
  6f6e74656e742e636f6d2f6f70656e61
  692f636861746770742d726574726965
  76616c2d706c7567696e2f6d61696e2f
  2e77656c6c2d6b6e6f776e2f61692d70
  6c7567696e2e6a736f6e
)
```

Once you run queries against it (i.e. ```dig @127.0.0.1 -p 1053 agent1._aid.example.com TYPE65400 +multiline```), you are given the correct output!

{% image "src/assets/img/posts/new-rr/dig-agent.png", "A screenshot of the dig agent output", "prose-img-md" %}

And optionally verify the same in the docker logs:

```bash

example.com.:1053
CoreDNS-1.14.1
linux/arm64, go1.24.13, d8f793b72
[INFO] 192.168.65.1:44630 - 12405 "A IN ns1.example.com. udp 44 false 4096" NOERROR qr,aa,rd 104 0.000414625s
[INFO] 192.168.65.1:57154 - 54431 "TYPE65400 IN agent1._aid.example.com. udp 52 false 4096" NOERROR qr,aa,rd 286 0.000190208s
```

So now, we are certain our DNS server is correctly providing answers, what now? We don’t get the pretty AID mnemonic or a nice textual RDATA syntax yet as we didn't change the parsing or encoding logic for RRs. Let's fix that. 😎

<br>

### More Invasive
IANA reserves 65280–65534 for “Private Use” RR TYPE values, which is exactly what we picked for our experimental record (65400). CoreDNS is heavily reliant on Miekg's DNS repository for RR serialization and processing, so it's time to wander into the land of innovation and importantly: sad times. The good news is Miekg's DNS fork provides a serialization string function to import custom names, zone files, and other types of records. Woohoo!

Let's work on creating our custom aidrr plugin:
```bash

mkdir -p plugin/aidrr
cat > plugin/aidrr/setup.go <<'EOF'
package aidrr

import (
    "github.com/coredns/caddy"
    "github.com/coredns/coredns/core/dnsserver"
    "github.com/coredns/coredns/plugin"
)

// We register as a normal CoreDNS plugin so the package is imported into the build,
// and init() (in aid.go) runs.
func init() { plugin.Register("aidrr", setup) }

func setup(c *caddy.Controller) error {
    c.Next() // consume "aidrr"
    dnsserver.GetConfig(c).AddPlugin(func(next plugin.Handler) plugin.Handler {
        return AidRR{Next: next}
    })
    return nil
}
EOF
```

Then the specific rr.go file:
```bash

cat > plugin/aidrr/aidrr.go <<'EOF'
package aidrr

import (
    "context"

    "github.com/coredns/coredns/plugin"
    "github.com/miekg/dns"
)

// AidRR is a no-op plugin whose only purpose is to ensure the package is linked
// and our init() registration of the AID RR type runs.
type AidRR struct {
    Next plugin.Handler
}

func (a AidRR) Name() string { return "aidrr" }

func (a AidRR) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) {
    return plugin.NextOrFailure(a.Name(), a.Next, ctx, w, r)
}
EOF
```

Now the rdata file:
``` bash

cat > plugin/aidrr/rdata.go <<'EOF'
package aidrr

import (
    "encoding/hex"
    "errors"
    "fmt"
    "strconv"
    "strings"

    "github.com/miekg/dns"
)

// Private-use RR type code (IANA range 65280-65534). We keep your existing 65400.
const TypeAID uint16 = 65400

// AIDRdata implements dns.PrivateRdata.
// Wire layout (v1) matches your earlier demo blob:
//
// version(1)
// flags(2)
// hash_algo(1)           1 = sha256
// agentid_len(1)         must be 32
// agentid(32)            sha256(canonical-json)
//
// uri_count(1)           currently 1
// uri_hash_algo(1)       1 = sha256
// uri_hash_len(1)        32
// uri_hash(32)           same as agentid (commit to JSON)
// uri_len(1)
// uri(uri_len bytes)
type AIDRdata struct {
    Version  uint8
    Flags    uint16
    HashAlgo uint8 // 1=sha256

    AgentID [32]byte

    URI string
}

// Register the mnemonic "AID" so zone files can use AID instead of TYPE65400.
// PrivateHandle wires type<->string mappings and the RR factory. [1](https://rfc-annotations.research.icann.org/rfc3597.html)[2](https://www.pentan.info/doc/rfc/3597.html)
func init() {
    dns.PrivateHandle("AID", TypeAID, func() dns.PrivateRdata { return new(AIDRdata) })
}

// String returns the zonefile presentation.
func (a *AIDRdata) String() string {
    return fmt.Sprintf("v=%d flags=%d hash=sha256:%x uri=%q",
        a.Version, a.Flags, a.AgentID[:], a.URI)
}

// Parse parses the RDATA tokens from the zonefile.
// We accept key=value tokens:
//   v=1 flags=0 hash=sha256:<64hex> uri="https://..."
// Also accept the URI as a trailing bare token if uri= isn't used.
func (a *AIDRdata) Parse(tokens []string) error {
    // Defaults
    a.Version = 1
    a.Flags = 0
    a.HashAlgo = 1

    var haveHash bool
    var haveURI bool

    for _, t := range tokens {
        if strings.Contains(t, "=") {
            kv := strings.SplitN(t, "=", 2)
            k := strings.ToLower(strings.TrimSpace(kv[0]))
            v := strings.TrimSpace(kv[1])

            switch k {
            case "v", "ver", "version":
                n, err := strconv.ParseUint(v, 10, 8)
                if err != nil {
                    return fmt.Errorf("AID: invalid version %q: %w", v, err)
                }
                a.Version = uint8(n)

            case "flags":
                n, err := strconv.ParseUint(v, 10, 16)
                if err != nil {
                    return fmt.Errorf("AID: invalid flags %q: %w", v, err)
                }
                a.Flags = uint16(n)

            case "hash":
                // Expect sha256:<hex>
                if !strings.HasPrefix(strings.ToLower(v), "sha256:") {
                    return fmt.Errorf("AID: unsupported hash format %q (want sha256:<hex>)", v)
                }
                h := v[len("sha256:"):]
                raw, err := hex.DecodeString(h)
                if err != nil || len(raw) != 32 {
                    return fmt.Errorf("AID: invalid sha256 hex (need 32 bytes): %q", h)
                }
                copy(a.AgentID[:], raw)
                haveHash = true

            case "uri":
                // lexer may already strip quotes; if not, trim them.
                a.URI = strings.Trim(v, "\"")
                haveURI = true
            }
        } else {
            // If it's not key=value, treat as URI if we haven't seen one.
            if !haveURI {
                a.URI = strings.Trim(t, "\"")
                haveURI = true
            }
        }
    }

    if !haveHash {
        return errors.New("AID: missing required hash=sha256:<hex>")
    }
    if !haveURI || a.URI == "" {
        return errors.New("AID: missing required uri=...")
    }
    return nil
}

// Len returns RDATA length in octets.
func (a *AIDRdata) Len() int {
    uriLen := len([]byte(a.URI))
    // fixed part: 1+2+1+1+32 + 1 + 1+1+32 +1 = 73 bytes, plus uri bytes
    return 73 + uriLen
}

// Copy copies receiver into dst.
func (a *AIDRdata) Copy(dst dns.PrivateRdata) error {
    o, ok := dst.(*AIDRdata)
    if !ok {
        return fmt.Errorf("AID: Copy dst has wrong type %T", dst)
    }
    *o = *a
    return nil
}

// Pack encodes RDATA into buf and returns bytes written.
func (a *AIDRdata) Pack(buf []byte) (int, error) {
    uriBytes := []byte(a.URI)
    if len(uriBytes) > 255 {
        return 0, fmt.Errorf("AID: URI too long (%d), must be <=255", len(uriBytes))
    }
    if a.Len() > len(buf) {
        return 0, errors.New("AID: buffer too small")
    }

    off := 0
    buf[off] = a.Version
    off++

    buf[off] = byte(a.Flags >> 8)
    buf[off+1] = byte(a.Flags)
    off += 2

    buf[off] = a.HashAlgo
    off++

    buf[off] = 32 // agentid_len
    off++

    copy(buf[off:off+32], a.AgentID[:])
    off += 32

    // uri_count
    buf[off] = 1
    off++

    // uri_hash_algo + uri_hash_len + uri_hash
    buf[off] = a.HashAlgo
    off++
    buf[off] = 32
    off++
    copy(buf[off:off+32], a.AgentID[:])
    off += 32

    // uri_len + uri
    buf[off] = byte(len(uriBytes))
    off++
    copy(buf[off:off+len(uriBytes)], uriBytes)
    off += len(uriBytes)

    return off, nil
}

// Unpack decodes RDATA from buf and returns bytes consumed.
func (a *AIDRdata) Unpack(buf []byte) (int, error) {
    if len(buf) < 73 {
        return 0, errors.New("AID: rdata too short")
    }
    off := 0

    a.Version = buf[off]
    off++

    a.Flags = uint16(buf[off])<<8 | uint16(buf[off+1])
    off += 2

    a.HashAlgo = buf[off]
    off++

    aidLen := int(buf[off])
    off++
    if aidLen != 32 || len(buf) < off+32 {
        return 0, errors.New("AID: invalid agentid length")
    }
    copy(a.AgentID[:], buf[off:off+32])
    off += 32

    uriCount := int(buf[off])
    off++
    if uriCount < 1 {
        return 0, errors.New("AID: uri_count must be >= 1")
    }

    // Read first URI entry only (v1)
    if len(buf) < off+1+1+32+1 {
        return 0, errors.New("AID: rdata too short for uri entry")
    }
    _ = buf[off] // uri_hash_algo
    off++
    uHashLen := int(buf[off])
    off++
    if uHashLen != 32 || len(buf) < off+32 {
        return 0, errors.New("AID: invalid uri_hash_len")
    }
    off += 32 // skip uri_hash

    uriLen := int(buf[off])
    off++
    if len(buf) < off+uriLen {
        return 0, errors.New("AID: uri_len beyond buffer")
    }
    a.URI = string(buf[off : off+uriLen])
    off += uriLen

    return off, nil
}
EOF
```

Edit plugin.cfg and add ```aidrr:aidrr``` to then pave the way to reload the plugins and containers:
```bash

make gen
make
docker build -t coredns-aid:local .
docker run --rm -it \
  -p 1053:1053/udp \
  -v "$PWD/Corefile":/Corefile \
  -v "$PWD/zones":/zones \
  coredns-aid:local \
  -conf /Corefile
```

Add the new AID record to the zonefile:
```zone

agent2._aid  IN  AID  v=1 flags=1 hash=sha256:354edc0a0f26dd973dd6c681338bb189bb42137697484d795b4b5d7d0426c1fb uri=https://raw.githubusercontent.com/openai/chatgpt-retrieval-plugin/main/.well-known/ai-plugin.json
```

Reload CoreDNS:
```bash

docker stop coredns-aid 2>/dev/null || true

docker run --rm -it \
  -p 1053:1053/udp \
  -v "$PWD/Corefile":/Corefile \
  -v "$PWD/zones":/zones \
  coredns-aid:local \
  -conf /Corefile
  ```

At this point you are now able to query your CoreDNS server for this new type of RR: AID!

> ```dig @127.0.0.1 -p 1053 agent2._aid.example.com TYPE65400 +noall +answer```

> ```[INFO] 192.168.65.1:32764 - 34282 "AID IN agent2._aid.example.com. udp 52 false 4096" NOERROR qr,aa,rd 286 0.00139425s```

<br>

# Future Work
The core identity (AgentID + capability bits) fits in under 128 bytes and the embedding gives a mathematically meaningful “shape” of the agent for selection/matching. The URI+hash tuples lets us keep all the giant stuff (full JSON, attestations, Merkle proofs) out of DNS while still cryptographically tied to this record. 

Because it’s a custom RR type, we can cleanly separate this from generic TXT/SVCB semantics that the BANDAID draft uses today. Though we are all proud of BANDAID and what it represents (use what exists today), it will be difficult to express all functionality as desired by SVCB paramaters. This density is desirable, but at some point we will run into a metadata-inclusion boundary.

What's left to do? 
- AID for identity + pointers (this demo)
- AIVECTOR for pure semantic vector (next)
- AIVECTOR RDATA (next)

Probably a client and AI agent to make use of the data, perhaps in discovery, and ideally I'm interested in the nearest-neighbor problem. 

# Appendix

<br>

## Links
[CoreDNS Fork](https://github.com/nicknacnic/coredns)