# FreeIran — Privacy, Censorship Resistance, Dark-Network Connectivity & v2rayN Research Report

**Project:** FreeIran — Autonomous Local Connectivity Engine
**Repository:** `github.com/Parsaetak/FreeIran`
**Current live branch:** `main`
**Verified live HEAD:** `8028f7f77d91c3d979d3308ae231729132bfe5ac`
**Current version:** `0.11.4`
**Research date:** 29 September 2026

[FreeIran live repository](https://github.com/Parsaetak/FreeIran?utm_source=chatgpt.com)
[FreeIran v0.11.4 commit](https://github.com/Parsaetak/FreeIran/commit/8028f7f77d91c3d979d3308ae231729132bfe5ac?utm_source=chatgpt.com)

---

## 1. Executive Summary

The original FreeIran objective was broader than simply providing a GUI around Xray or sing-box.

The actual engineering objective is:

> **Create a Windows-first connectivity system that remains usable under censorship, minimizes network and device-identifying leakage, automatically adapts to changing network conditions, and verifies its own protection rather than merely claiming that a tunnel is active.**

The research shows that this is technically achievable as a **layered privacy and censorship-resistance system**, but not as a literal guarantee that a powerful authority can never identify a particular user.

That distinction matters because the network contains several independent identification surfaces:

```text
Local Wi-Fi / LAN
        ↓
ISP / access provider
        ↓
Censorship / DPI infrastructure
        ↓
VPN / proxy / Tor entry
        ↓
Exit / destination
        ↓
Website / application / account
        ↓
Local operating system / device
```

A VPN solves only part of this chain.

A serious FreeIran design therefore needs several independent layers:

```text
FULL TUN
+
DNS LEAK PREVENTION
+
IPv4 / IPv6 LEAK CONTROL
+
WFP KILL SWITCH
+
ADAPTIVE TRANSPORT SELECTION
+
REALITY / XHTTP
+
HYSTERIA2 / TUIC / SHADOWSOCKS
+
WIREGUARD / AMNEZIAWG
+
TOR + PLUGGABLE TRANSPORTS
+
ONION-ONLY MODE
+
OPTIONAL I2P
+
ROUTING ENGINE
+
MULTI-HOP
+
DEVICE-ID MINIMIZATION
+
PRIVACY DIAGNOSTICS
+
CONTINUOUS CONNECTION PROOF
```

The central conclusion is that FreeIran should **not** become a larger collection of protocol parsers. It should become a unified **connection, privacy, routing, and evidence system** around the protocol engines it already manages.

---

# 2. Current FreeIran Baseline

The live `main` branch at `8028f7f77d91c3d979d3308ae231729132bfe5ac` is version `0.11.4`.

The current architecture already contains many of the foundations required for the long-term design:

```text
React / TypeScript UI
        ↓
Wails
        ↓
Go application orchestration
        ↓
Discovery / Storage / Testing / Ranking
        ↓
Core registry + core manager
        ↓
Xray / V2Ray / sing-box
        ↓
Shared process supervision
```

FreeIran already has:

* canonical protocol configuration;
* Xray, V2Ray and sing-box integration;
* verified core acquisition;
* process supervision;
* deterministic core selection;
* configuration testing;
* measured latency and URL testing;
* failure classification;
* candidate ranking;
* connection recovery;
* system proxy support;
* real Windows TUN support;
* route and interface verification;
* crash-recovery state;
* Tor and Psiphon provider abstractions;
* bounded runtime state;
* security scanning;
* Windows CI;
* explicit evidence-oriented connection state.

The architecture therefore does not need to be replaced.

It needs to be extended from:

```text
"connect to a proxy"
```

into:

```text
"establish, prove, protect, monitor and recover an end-to-end network session."
```

---

# 3. The Real Threat Model

“Hide my IP” is not a complete threat model.

FreeIran should explicitly model at least seven observation layers.

## 3.1 Local network

A Wi-Fi access point or local network can observe things such as:

* the local device's network-layer activity;
* MAC address;
* association information;
* timing;
* packet volume;
* potentially the fact that the device is communicating with a particular remote endpoint.

Windows supports randomized hardware addresses for Wi-Fi networks, which can reduce link-layer tracking, but this is fundamentally different from changing the physical hardware identifier.

### FreeIran requirement

FreeIran should provide a **Local Link Privacy Audit**:

```text
Wi-Fi hardware address:
    randomized / stable / unknown

Local network:
    visible to local network

Physical MAC:
    never transmitted by FreeIran

Device serial / UUID:
    not collected by FreeIran
```

FreeIran should not claim that a VPN makes the MAC invisible to the local access point.

---

# 4. IP Privacy

A VPN normally changes what the destination sees:

```text
Without VPN:

User → Website
Website sees user's public IP

With VPN:

User → VPN → Website
Website sees VPN exit IP
```

The ISP can still see that the subscriber's connection is communicating with the VPN endpoint, while the VPN endpoint can generally observe the originating connection and metadata.

Cloudflare describes the same trust shift: the ISP sees the encrypted VPN connection while the destination sees the VPN server rather than the client's original IP.

EFF likewise describes a VPN as moving trust from the ISP toward the VPN provider rather than magically eliminating trust.

### FreeIran requirement

The application should distinguish:

```text
Local IP
Public IPv4
Public IPv6
VPN exit IP
Observed country
Observed ASN
Route status
```

and verify them after connection.

A displayed “Connected” state must never itself imply that the public IP has changed.

---

# 5. DNS Is a First-Class Security Boundary

A tunnel can be functioning while DNS still escapes through the physical interface.

This is particularly important under censorship because DNS filtering can be performed before any encrypted application connection begins.

Current sing-box documentation provides:

* TUN `auto_route`;
* `strict_route`;
* `auto_detect_interface`;
* DNS hijacking;
* native DNS handling;
* route-level DNS behavior;
* Windows-specific leak prevention.

With sing-box TUN, `strict_route` combined with DNS hijacking can block DNS traffic on interfaces other than the TUN interface.

Current sing-box route rules also support domain, IP-CIDR, process and rule-set based matching.

### FreeIran design

DNS should become a separate engine:

```text
engine/dns/
    resolver
    bootstrap
    encrypted_transport
    routing
    leak_test
    cache
    evidence
```

Potential modes:

```text
Tunnel DNS
Encrypted DNS
Bootstrap DNS
Local Direct DNS
Fake-IP DNS
Split DNS
DNS Block
DNS Fail-Closed
```

The important architectural principle is:

> **DNS resolution policy must be decided centrally, not independently by every protocol core.**

---

# 6. IPv6 Must Be Treated as Equal to IPv4

If the VPN protects IPv4 but leaves IPv6 outside the tunnel:

```text
IPv4 → VPN
IPv6 → ISP
```

the application has a major leak.

Recent v2rayN releases themselves have had to modify IPv6 TUN behavior, including changes designed to make IPv6 follow the TUN route and changes around AAAA handling.

### FreeIran requirement

The connection proof must explicitly contain:

```go
IPv4Verified bool
IPv6Verified bool
```

and support three honest outcomes:

```text
IPv6 protected
IPv6 unavailable
IPv6 leak detected
```

“IPv6 disabled” and “IPv6 protected” are not the same state.

---

# 7. Kill Switch: The Major Missing Protection Layer

Current FreeIran TUN routing is not a kill switch.

Routing answers:

> Where should a packet go?

A kill switch answers:

> Is a packet allowed to leave the machine at all?

Windows Filtering Platform (WFP) provides the kernel networking filter infrastructure needed to implement packet filtering policies.

WireGuard for Windows provides a useful reference design: its kill switch creates a narrowly defined allow policy for tunnel traffic and required local services while blocking unrelated outbound traffic.

### FreeIran requirement

Implement:

```text
engine/firewall/
    wfp_windows.go
    policy.go
    state.go
    recovery.go
    rules.go
    diagnostics.go
```

The kill-switch lifecycle should be:

```text
DIRECT
  ↓
ARMING
  ↓
BOOTSTRAP-ALLOW
  ↓
TUN ESTABLISHED
  ↓
PROTECTED-ALLOW
  ↓
MONITORING
```

On failure:

```text
CORE CRASH
     ↓
TUN LOSS
     ↓
VERIFY FAILURE
     ↓
PROTECTED TRAFFIC BLOCKED
     ↓
RECOVERY
```

The critical property is:

```text
Tunnel lost ≠ traffic silently falls back to direct Internet
```

The WFP policy must also survive unexpected process death and be safely recoverable after restart.

---

# 8. Censorship Is Not Static

Current Iranian censorship research shows why a fixed “best protocol” architecture is insufficient.

A 2026 USENIX Security study based on leaked Geedge/Terrestrial Security Gateway material demonstrates that sophisticated commercial DPI systems can recognize and classify multiple protocols and maintain protocol fingerprints.

Research into Iran's June 2025 shutdown found service-level and progressive restrictions affecting a very large portion of Iranian network infrastructure rather than merely one simple BGP withdrawal mechanism.

The 2026 environment has also shown periods in which DNS, UDP/QUIC, TLS and application-level access behaved differently.

That means:

> **The correct FreeIran abstraction is not “best protocol.” It is “best currently viable path under observed conditions.”**

---

# 9. Failure Differential Measurement

FreeIran's existing failure classification can be expanded into a censorship-specific evidence ladder:

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
Handshake
 ↓
HTTP
 ↓
End-to-end Internet
```

For example:

```text
DNS succeeds
TCP succeeds
TLS ClientHello sent
TLS handshake repeatedly times out
```

This is materially different from:

```text
DNS fails
```

OONI has documented Iranian measurements where TCP connections to WhatsApp succeeded while TLS handshakes stalled after ClientHello, consistent with filtering at a later stage of the connection.

FreeIran should therefore record:

```go
FailureStage
FailureClass
ObservedBytes
RemoteEndpoint
Transport
SNI
Timing
RetryCount
Environment
```

without storing unnecessary sensitive payload data.

---

# 10. REALITY

Xray's REALITY system is specifically designed to make a connection resemble legitimate TLS traffic to a real server rather than expose a plainly recognizable proxy protocol.

Xray documents REALITY support with RAW, XHTTP and gRPC transports.

### FreeIran status

FreeIran already supports REALITY through its Xray and sing-box integrations.

The next step is not merely “add REALITY.”

The next step is:

```text
REALITY becomes one candidate in an adaptive transport engine.
```

The engine should determine whether a REALITY endpoint is currently usable from the detected environment.

---

# 11. XHTTP

XHTTP is important because modern censorship resistance increasingly depends on making the outer transport look like ordinary HTTPS traffic rather than exposing a distinctive VPN signature.

Current Xray documentation lists XHTTP among its modern transport mechanisms.

v2rayN's current source also explicitly models XHTTP as a transport and generates corresponding configuration fields.

### FreeIran recommendation

Make transport a first-class capability:

```text
TCP
WebSocket
gRPC
HTTPUpgrade
XHTTP
QUIC
```

rather than burying transport behavior inside protocol adapters.

---

# 12. Hysteria2 and TUIC

Hysteria2 and TUIC are useful because they use UDP/QUIC-style transport and can perform differently from TCP-based transports.

However:

```text
UDP available
    ≠
UDP always useful
```

Some restricted networks heavily block UDP or QUIC.

Therefore they should be adaptive candidates rather than universal defaults.

FreeIran already has Hysteria2, TUIC and WireGuard semantics in the canonical configuration and sing-box backend.

---

# 13. Shadowsocks

Shadowsocks remains useful as an alternative encrypted proxy family.

The important design lesson is that Shadowsocks is another candidate transport, not a guaranteed anti-censorship solution.

Recent community measurements around Iran reported continued viability for some Shadowsocks configurations after other techniques became less reliable. These findings are measurement-specific rather than universal, so FreeIran should treat them as inputs to probing rather than permanent protocol guarantees.

---

# 14. Fragmentation Should Not Become the Foundation

TLS fragmentation is an example of a technique that can work under one censoring configuration and fail after the censor adapts.

Community measurements reported that Iran began defeating some fragmented TLS ClientHello approaches in July 2026.

The engineering lesson is stronger than the individual result:

> **Do not make FreeIran dependent on one clever packet manipulation trick.**

Fragmentation may remain an optional transport tactic, but it should never be the central censorship-resistance layer.

---

# 15. AmneziaWG

AmneziaWG deserves first-class consideration.

It is based on WireGuard concepts but modifies identifiable traffic properties. Current Amnezia documentation describes successive generations designed to alter headers, packet sizes, sequences, timing and related characteristics to resist statistical identification.

The official Windows implementation includes an embeddable tunnel library and Wintun-based client components.

### Integration model

Do not integrate the AmneziaWG GUI.

Do not create a second independent process-management architecture.

Instead:

```text
FreeIran canonical WireGuard config
        ↓
WireGuard / AmneziaWG variant
        ↓
official embeddable Windows implementation
        ↓
FreeIran lifecycle / supervision / verification
```

The privileged Windows implementation deserves a dedicated security review because the official Windows client contains privileged services and IPC components.

### Canonical model

A suitable extension is:

```text
WireGuardVariant:
    standard
    amneziawg
```

while retaining shared:

```text
PrivateKey
PublicKey
Endpoint
AllowedIPs
InterfaceAddress
DNS
MTU
Keepalive
```

and adding variant-specific obfuscation parameters only when required.

---

# 16. Tor Is the Most Important “Dark Network” Addition

Tor is not simply another VPN protocol.

Its trust topology is different:

```text
User
 ↓
Guard
 ↓
Middle
 ↓
Exit
 ↓
Internet
```

For an onion service:

```text
User
 ↓
Tor circuit
 ↓
Rendezvous
 ↓
Onion service
```

There is no normal Internet exit in the onion-service path.

Tor supports several pluggable transports designed for censored networks, including:

* obfs4;
* Snowflake;
* WebTunnel;
* meek.

Tor Project reporting in 2025 specifically identified Snowflake as the most-used network traffic obfuscation mechanism in Iran at that time.

---

# 17. Tor Modes for FreeIran

FreeIran should not expose only a generic “Tor” button.

It should model explicit connection modes.

### Tor Circumvention

```text
FreeIran
  ↓
Tor
  ↓
Pluggable Transport
  ↓
Internet
```

### Onion Only

```text
FreeIran
  ↓
Tor
  ↓
.onion
```

No ordinary exit traffic.

This makes the trust model much clearer and provides a proper dark-network capability.

### Tor transport options

```text
Automatic
obfs4
Snowflake
WebTunnel
meek
```

The selection logic should be evidence-driven.

FreeIran should not hard-code one bridge set as though it will remain functional indefinitely.

---

# 18. Onion-Only Mode

This should be a genuine security mode, not merely a UI shortcut.

Requirements:

```text
Only .onion destinations
No ordinary public Internet fallback
No accidental system-proxy escape
Dedicated DNS behavior
Dedicated process profile
Dedicated browser/app state where applicable
Independent connection proof
```

The purpose is to make the trust topology explicit.

A website reached through a Tor onion service does not receive a normal VPN exit IP in the ordinary manner because the service is reached through Tor's onion architecture.

Tor's onion-service design uses introduction points, signed descriptors and rendezvous mechanisms rather than directly exposing the service's physical network location.

---

# 19. I2P

I2P is another important dark-network technology, but it should not be represented as a generic replacement for Tor or an ordinary VPN.

The I2P project itself describes I2P as primarily an internal anonymous network, with limited outproxy functionality rather than a general Internet anonymity service.

I2P SOCKS can provide application access to I2P resources, but general Internet outproxy use has different trust and reliability characteristics.

### FreeIran integration

Create:

```text
I2P Provider
```

with an explicit mode:

```text
I2P-only
```

for:

```text
.i2p
```

services.

Do not make:

```text
FreeIran → I2P → arbitrary public Internet
```

the default.

---

# 20. Multi-Hop

FreeIran can also support multi-hop routing:

```text
User
 ↓
Hop 1
 ↓
Hop 2
 ↓
Internet
```

or:

```text
User
 ↓
VPN
 ↓
Tor
 ↓
Internet
```

or:

```text
User
 ↓
Tor
 ↓
VPN
 ↓
Internet
```

However, each topology changes who can observe what.

Multi-hop should therefore be represented as a **trust topology**, not advertised simply as “more anonymous.”

FreeIran should display:

```text
Local observer
Hop 1
Hop 2
Exit
Destination
```

so the user understands which party can observe which side of the connection.

---

# 21. Proxy Chains

v2rayN already contains group and proxy-chain concepts, including selector/urltest-style behavior and chain outbounds in its sing-box configuration generator.

This is worth adopting conceptually.

But FreeIran should preserve one authority:

```text
Connection Engine
      ↓
Route Plan
      ↓
Core Adapter
```

not:

```text
Core A has its own routing logic
Core B has a different routing logic
Tor has another connection engine
```

---

# 22. Routing Intelligence

FreeIran needs a canonical routing engine independent of Xray or sing-box.

Suggested module:

```text
engine/routing/
    model.go
    matcher.go
    compiler.go
    ruleset.go
    resolver.go
    dns.go
    profiles.go
```

Canonical rule fields:

```text
Domain
DomainSuffix
DomainRegex
IPCIDR
Port
Protocol
ProcessName
ProcessPath
Action
Outbound
DNSPolicy
```

Actions:

```text
Direct
Proxy
Reject
Block
Tor
I2P
```

---

# 23. Practical Routing Profiles

A mature UI can provide:

```text
Global Proxy
Iran Direct + Global Proxy
Global Direct
Block
Tor
I2P
Custom
```

“Iran Direct + Global Proxy” means:

```text
Iranian / domestic services → Direct
Everything else → Proxy
```

This is a routing policy, not a promise that domestic routes are always reachable.

Community repositories such as `routing-vpn-iran` and Iranian geo-routing projects provide useful source material for `.ir`, banking, payment and domestic-service routing rules.

These rules should be treated as **untrusted update inputs** and versioned/hashed rather than embedded as unquestioned security truth.

---

# 24. DNS + Routing Must Be Integrated

The route engine should not decide only:

```text
domain → proxy
```

It should also decide:

```text
where is that domain resolved?
which resolver?
through which transport?
with which bootstrap path?
```

A modern rule may therefore look conceptually like:

```text
google.com
    route = proxy
    dns = tunneled-doh
```

while:

```text
bank.example.ir
    route = direct
    dns = domestic
```

or:

```text
example.onion
    route = tor
    dns = tor
```

This prevents routing and DNS from disagreeing.

---

# 25. The Device-Identifier Problem

“Manufacturer number” is ambiguous.

There are at least three possibilities.

## Public IP

A network identity.

VPN/TUN can change what destinations see.

## MAC / OUI

A local link-layer identifier.

The OUI portion can identify a registered hardware vendor.

This is primarily visible on the local network.

## SMBIOS serial / UUID

Windows/firmware-level device identifiers.

Microsoft's SMBIOS documentation identifies serial number and UUID properties as hardware identification information.

A VPN cannot rewrite these identifiers for the operating system.

### FreeIran policy

FreeIran should:

```text
NOT collect serial number
NOT collect SMBIOS UUID
NOT collect motherboard identifier
NOT collect disk serial
NOT collect CPU serial-equivalent identifiers
NOT transmit these to a server
```

unless the user explicitly enables a diagnostic function that genuinely needs one.

The stronger privacy architecture is **data minimization**, not pretending the identifiers have been changed.

---

# 26. Browser Fingerprinting

Even with:

```text
VPN IP
+
Tor
+
TUN
+
DNS protection
```

a browser can still expose identifying properties through:

* cookies;
* account login;
* local storage;
* screen characteristics;
* fonts;
* APIs;
* browser configuration;
* behavioral patterns;
* application-specific identifiers.

Therefore:

```text
network anonymity
≠
browser anonymity
```

FreeIran should expose this limitation directly.

An eventual **Privacy Browser Mode** could become a separate project layer, but the VPN engine itself should not claim to solve browser fingerprinting.

---

# 27. External IP APIs Need Care

v2rayN currently performs public-IP checks through configurable external IP APIs, including `ip.sb`-type endpoints. Its `ConnectionHandler` measures ping and then obtains public IP information through an external API via the local proxy.

This is useful functionally, but it also means:

```text
Privacy check
      ↓
third-party request
      ↓
third-party logging / metadata risk
```

### FreeIran should improve this

Support:

```text
Project-controlled verification service
Self-hosted verification service
Multiple trusted endpoints
Offline / unavailable result
```

and make the distinction explicit:

```text
Verified
Not verified
Unavailable
```

rather than manufacturing confidence.

---

# 28. v2rayN Deep Source Findings

The current v2rayN source is useful because it demonstrates several mature architectural patterns that FreeIran should understand.

The current core enumeration includes:

```text
Xray
mihomo
sing-box
Hysteria
NaiveProxy
TUIC
Juicity
Hysteria2
Brook
overtls
ShadowQUIC
Mieru
...
```

and configuration types include:

```text
VMess
VLESS
Trojan
Shadowsocks
Hysteria2
TUIC
WireGuard
HTTP
Anytls
Naive
ProxyChain
PolicyGroup
```

This demonstrates the benefit of treating “core” and “configuration protocol” as separate dimensions.

FreeIran already follows this principle conceptually.

---

# 29. v2rayN TUN Architecture

The current source has an explicit `TunModeItem` containing:

```text
EnableTun
AutoRoute
StrictRoute
Stack
MTU
IPv6
ICMP routing
route exclusions
IPv4 address
IPv6 address
```

and separate configuration generation for Xray and sing-box.

Its sing-box implementation uses:

```text
auto_route
strict_route
stack
IPv4
IPv6
route_exclude_address
auto_detect_interface
DNS hijacking
```

The current source also contains a TUN DNS profile that can send remote DNS through the proxy and local DNS directly.

This is a useful reference model for FreeIran.

---

# 30. Important v2rayN Lessons From Real Bugs

The reason not to simply copy v2rayN architecture is that its increasing feature surface has created real TUN/configuration failure modes.

Recent reports include:

* Xray TUN DNS leak problems;
* sing-box TUN DNS behavior differences;
* domain-strategy mismatches between Xray and sing-box;
* routing/rule-set bootstrap deadlocks;
* rule-generation mistakes in sing-box configurations.

One reported class of issue involved a generated sing-box configuration using a strategy value appropriate to another core, producing a TUN-related failure.

Another involved rule-set bootstrap traffic becoming entangled with the proxy route itself, producing a deadlock-like configuration failure.

### Architectural conclusion

FreeIran's existing philosophy is correct:

```text
one canonical model
        ↓
core-specific compiler
```

rather than:

```text
separate product logic for Xray
separate product logic for sing-box
separate product logic for Tor
```

---

# 31. v2rayN Process Supervision

v2rayN uses a Windows Job-based process-management model around core processes.

Its current `CoreManager`:

* creates/configures core processes;
* starts them through a common process service;
* waits for proxy readiness;
* attaches Windows processes to a Job;
* stops them during lifecycle changes.

That is an important pattern for FreeIran.

FreeIran already has its own unified process supervision and should retain that architectural advantage rather than reproduce v2rayN's singleton-style lifecycle management.

---

# 32. v2rayN Proxy/PAC Architecture

v2rayN also supports:

```text
system proxy
PAC
proxy exceptions
local proxy ports
```

and runs a local PAC service when PAC mode is selected.

Its `SysProxyHandler` changes the system proxy and its `PacManager` serves locally generated PAC content.

This corresponds closely to the current FreeIran system-proxy implementation.

The major conceptual difference is that FreeIran can place this under a single verified session lifecycle rather than making proxy state one of several semi-independent runtime systems.

---

# 33. v2rayN URL/IP Verification

v2rayN's `ConnectionHandler`:

```text
proxy
 ↓
HTTP request
 ↓
public IP lookup
 ↓
country/IP result
```

is a useful reference.

FreeIran should go farther by making verification multi-dimensional:

```text
Core Ready
+
TUN Active
+
Route Observed
+
DNS Verified
+
IPv4 Verified
+
IPv6 Verified
+
Public IP Verified
+
Traffic Verified
+
Kill Switch Armed
```

---

# 34. Current FreeIran Connection Model Should Evolve

The existing state machine is already approximately:

```text
Disconnected
→ Selecting
→ Preparing
→ StartingCore
→ WaitingForReady
→ Connected
→ Verifying
→ ConnectedVerified
```

That should evolve toward:

```text
IDLE
 ↓
DETECTING
 ↓
PREPARING
 ↓
CONNECTING
 ↓
CORE_READY
 ↓
ROUTE_READY
 ↓
DNS_READY
 ↓
TRAFFIC_VERIFIED
 ↓
PROTECTED
 ↓
MONITORING
 ↓
DEGRADED
 ↓
REPAIR / RECONNECT / TRANSPORT_SWITCH / BACKEND_SWITCH
```

“Protected” should become a proof-backed state.

---

# 35. Connection Proof

Recommended model:

```go
type ConnectionProof struct {
    CoreReady       bool
    TunnelInterface bool
    RouteObserved   bool
    DNSObserved     bool
    IPv4Verified    bool
    IPv6Verified    bool
    PublicIPVerified bool
    TrafficVerified bool
    KillSwitchArmed bool
    VerifiedAt      time.Time
}
```

FreeIran should never publish:

```text
Protected
```

until the required evidence is satisfied.

The UI should instead be able to say:

```text
Connected
but IPv6 not verified
```

or:

```text
Tunnel active
but Internet verification unavailable
```

---

# 36. Adaptive Connection Engine

The central future architecture should be:

```text
Environment Detector
        ↓
Cheap Probes
        ↓
Candidate Generation
        ↓
Transport/Core Compatibility
        ↓
Bounded Connection Attempts
        ↓
End-to-End Verification
        ↓
Evidence Recording
        ↓
Ranking
        ↓
Adaptive Selection
```

A candidate should be described by:

```text
Protocol
Transport
Core
Server IP
Server ASN
Port
SNI
TLS fingerprint
REALITY parameters
Provider
Historical success
Recent failure
Freshness
Environment
```

---

# 37. Endpoint-Diversity Intelligence

A particularly important improvement is to understand that several configurations may secretly represent the same infrastructure.

For example:

```text
Node A → 1.2.3.4:443
Node B → same server
Node C → same ASN
Node D → same transport + same endpoint family
```

Trying all four after the same infrastructure failure is wasted time.

FreeIran should maintain an infrastructure fingerprint such as:

```text
ServerIP
ASN
Port
Transport
SNI
Provider
ProtocolFamily
```

and use it during recovery.

This is not static blacklisting.

It is evidence-aware exploration.

---

# 38. Censor-Behavior Fingerprints

FreeIran can eventually maintain a local environmental fingerprint such as:

```text
DNS: reachable
TCP/443: reachable
TLS: intermittent
UDP/443: blocked
HTTP: reachable
XHTTP: successful
REALITY: successful
Snowflake: successful
```

This allows the application to remember:

```text
"This environment is currently hostile to UDP."
```

without pretending that the result applies universally.

---

# 39. Privacy Modes

The mature FreeIran UI should expose explicit modes instead of one generic “VPN” setting.

## Protected Mode

```text
Full TUN
DNS protection
IPv4/IPv6 protection
WFP kill switch
End-to-end verification
Continuous monitoring
```

## Censorship Adaptive

```text
Automatic transport probing
REALITY
XHTTP
Hysteria2
TUIC
Shadowsocks
WireGuard
AmneziaWG
```

## Tor Circumvention

```text
Tor
obfs4
Snowflake
WebTunnel
meek
```

## Onion Only

```text
Tor
.onion destinations
no public exit fallback
```

## I2P

```text
I2P network
.i2p destinations
```

## Multi-Hop

```text
two or more independent hops
explicit trust diagram
```

## Low-Metadata Mode

```text
No telemetry
Minimal logs
No unnecessary external IP checks
No hardware-ID collection
Updates only through defined policy
```

---

# 40. “Invisible Mode” Should Become a Technical Contract

The original product ambition can remain as the long-term objective, but the implementation should express it through measurable guarantees.

Instead of:

```text
"You are invisible."
```

FreeIran should provide a privacy contract such as:

```text
Original public IPv4 hidden from destination: VERIFIED
Original public IPv6 hidden from destination: VERIFIED
DNS outside tunnel: NOT OBSERVED
Tunnel route active: VERIFIED
Kill switch: ARMED
Application hardware identifiers transmitted: NONE
Local MAC: visible to local network
Browser fingerprint: outside network engine control
VPN endpoint can observe source connection: YES
Global traffic correlation resistance: NOT GUARANTEED
```

That is substantially more useful than a marketing claim.

---

# 41. Advanced “Low-Linkability” Architecture

A long-term architecture could combine:

```text
Full TUN
+
WFP
+
Tor / VPN / AmneziaWG diversity
+
Multi-hop
+
DNS protection
+
IPv6 protection
+
No hardware IDs
+
Minimal logs
+
Dedicated privacy browser
+
Separate application profiles
+
Onion-only capability
```

The goal is reducing the number of independent observations that can be correlated.

It does **not** eliminate correlation against a sufficiently powerful observer.

---

# 42. Why Dark Networks Matter

Dark-network integration is not merely a novelty.

It provides another network topology.

Ordinary VPN:

```text
User → VPN → Website
```

Tor:

```text
User → Guard → Middle → Exit → Website
```

Onion:

```text
User → Tor → Rendezvous → Onion service
```

I2P:

```text
User → I2P overlay → I2P destination
```

Each architecture has a different trust and performance profile.

Therefore FreeIran can become an **orchestrator of network topologies**, rather than simply a proxy client.

---

# 43. Emergency Connectivity Philosophy

During large-scale shutdowns, there may be no reachable foreign VPN endpoint at all.

Recent Iranian shutdown research has shown periods of extremely severe connectivity loss, including near-total nationwide disruption. Access Now reported more than a 98% connectivity drop during the February 2026 shutdown.

IODA measurements have also documented major Iranian connectivity collapses.

Therefore:

```text
No reachable route
```

may actually mean:

```text
No viable Internet path exists
```

rather than:

```text
FreeIran selected the wrong protocol.
```

This distinction should be visible to the user.

---

# 44. Recovery Logic

Recovery should classify the problem before changing protocols.

Example:

```text
DNS failure
→ alternate DNS/bootstrap

TCP failure
→ alternate endpoint / port

TLS/SNI failure
→ transport switch

UDP blocked
→ TCP-based transport

Core crash
→ same backend retry

Core repeatedly fails
→ backend switch

Route failure
→ TUN repair

DNS works but traffic fails
→ routing/compiler diagnosis
```

This is much more useful than simply trying nodes in a fixed list.

---

# 45. Core/Provider Architecture

FreeIran currently has:

```text
Xray
V2Ray
sing-box
Tor
Psiphon
```

The system should eventually have a unified provider contract:

```text
Resolve
Install
Verify
Prepare
Start
Ready
Connect
Health
Monitor
Recover
Stop
Cleanup
```

Every provider/core should expose the same lifecycle semantics.

This lets the Connection Engine remain independent of the underlying implementation.

---

# 46. The Most Important Missing FreeIran Components

Relative to the long-term objective, the major missing systems are:

```text
1. WFP kill switch
2. First-class DNS engine
3. IPv6 leak enforcement
4. Privacy exposure diagnostics
5. Adaptive transport engine
6. AmneziaWG
7. Tor pluggable transports
8. Onion-only mode
9. I2P mode
10. Canonical routing engine
11. Infrastructure fingerprinting
12. Censorship differential probes
13. ConnectionProof
14. Low-metadata mode
15. Optional multi-hop topology
```

These should be implemented on top of the current architecture.

---

# 47. Current Stale/Consistency Issues

The live FreeIran tree also contains consistency issues that should be cleaned before a major networking expansion.

The most obvious one is:

```text
engine/tunnel/tunnel.go
```

whose historical package comment still describes TUN as disabled even though the implementation is now a real sing-box/Wintun TUN backend.

There are also stale references to sing-box `1.14.0` in comments/docs/tests even though v0.11.4 has aligned the active pin and CI to `1.14.1`.

These are not necessarily runtime defects, but they create dangerous ambiguity for maintainers and AI agents.

The rule should be:

```text
Current behavior → current documentation
Historical behavior → explicitly historical
```

---

# 48. Recommended Engineering Roadmap

## Phase 1 — Full Tunnel Correctness

Implement:

```text
TUN
+
DNS engine
+
IPv4 proof
+
IPv6 proof
+
route proof
+
traffic proof
+
WFP kill switch
+
crash recovery
```

This should be the next major foundation.

---

## Phase 2 — Adaptive Censorship Engine

Integrate:

```text
DNS/TCP/TLS/HTTP differential probes
+
REALITY
+
XHTTP
+
Hysteria2
+
TUIC
+
Shadowsocks
+
WireGuard
+
AmneziaWG
```

Selection must be evidence-driven.

---

## Phase 3 — Tor Integration

Add:

```text
Tor provider
obfs4
Snowflake
WebTunnel
meek
```

and:

```text
Tor Circumvention
Onion Only
```

modes.

---

## Phase 4 — Routing Intelligence

Add:

```text
Direct
Proxy
Block
Tor
I2P
Custom
```

with central compilation into Xray/sing-box.

---

## Phase 5 — Device/Privacy Layer

Add:

```text
Privacy Exposure
MAC/randomization audit
hardware-ID collection audit
DNS exposure
IPv6 exposure
proxy ownership
kill-switch status
external IP verification
```

---

## Phase 6 — Advanced Topologies

Add:

```text
Multi-hop
Tor-over-VPN
VPN-over-Tor
I2P-only
application-specific routing
```

with explicit trust diagrams.

---

# 49. Testing Requirements

Networking features must be tested at the level where failures actually happen.

## Unit

```text
routing
DNS
failure classification
candidate scoring
connection proofs
WFP policy generation
state transitions
```

## Integration

```text
core startup
TUN creation
route ownership
DNS hijack
public IP verification
IPv6 behavior
process crash
reconnect
rollback
```

## Windows

```text
Wintun
IP Helper APIs
WFP
WinINet
adapter lifecycle
route lifecycle
power-cycle/restart recovery
```

## Real Core

Test against:

```text
Xray
sing-box
V2Ray
AmneziaWG
Tor
```

where licensing, binaries and test infrastructure permit.

---

# 50. Clean-Room Verification

The final acceptance test should be stronger than:

```text
application opens
```

A clean machine should verify:

```text
Install
→ Core acquisition
→ Checksum verification
→ Tunnel
→ DNS
→ Route
→ IPv4
→ IPv6
→ Internet
→ Kill switch
→ Core crash
→ Automatic recovery
→ Disconnect
→ No orphan process
→ No leaked proxy
→ No stale firewall policy
→ Clean uninstall
```

---

# 51. Release Acceptance Standard

A release intended to replace v2rayN for serious daily use should only be considered complete when:

### Connectivity

```text
Quick Connect
→ fresh evidence
→ ranked candidates
→ connection
→ end-to-end verification
```

### Protection

```text
TUN
DNS
IPv4
IPv6
Kill switch
```

all produce observable evidence.

### Censorship

```text
adaptive transport selection
```

must respond to actual failures.

### Dark Network

```text
Tor
Snowflake
obfs4
WebTunnel
onion
I2P
```

must have explicit, testable lifecycle states.

### Recovery

```text
core crash
transport failure
route failure
DNS failure
```

must trigger bounded recovery rather than silent degradation.

### Privacy

```text
No hardware identifiers
No unnecessary telemetry
No fake anonymity claims
```

must be architectural properties.

---

# 52. Important Non-Goals

FreeIran should **not** claim any of the following without extraordinary evidence:

```text
"Completely invisible"
"Impossible to detect"
"Untraceable"
"Authorities cannot identify the user"
"Hardware identity is changed"
"Browser fingerprint is hidden"
"Global traffic correlation is impossible"
```

The technically defensible objective is:

```text
Maximize resistance to censorship
Minimize network exposure
Minimize linkability
Prevent accidental leakage
Reduce unnecessary metadata
Provide multiple independent connectivity paths
Verify protection continuously
```

---

# 53. Final Architecture

The long-term FreeIran architecture should become:

```text
FREEIRAN
│
├── CONNECTION ENGINE
│   ├── Environment Detection
│   ├── Candidate Selection
│   ├── Adaptive Transport
│   ├── Session Lifecycle
│   └── Recovery
│
├── PRIVACY ENGINE
│   ├── DNS
│   ├── IPv4
│   ├── IPv6
│   ├── WFP Kill Switch
│   ├── Leak Tests
│   └── Privacy Exposure
│
├── TRANSPORT ENGINE
│   ├── Xray
│   │   ├── REALITY
│   │   ├── XHTTP
│   │   └── TCP/WS/gRPC
│   ├── sing-box
│   │   ├── Hysteria2
│   │   ├── TUIC
│   │   ├── WireGuard
│   │   └── TUN
│   ├── V2Ray
│   ├── AmneziaWG
│   ├── Tor
│   │   ├── obfs4
│   │   ├── Snowflake
│   │   ├── WebTunnel
│   │   └── meek
│   └── I2P
│
├── ROUTING ENGINE
│   ├── Direct
│   ├── Proxy
│   ├── Block
│   ├── Tor
│   ├── I2P
│   └── Custom
│
├── EVIDENCE ENGINE
│   ├── ConnectionProof
│   ├── Failure Classification
│   ├── Route Evidence
│   ├── DNS Evidence
│   ├── IP Evidence
│   └── Health History
│
├── CORE / PROVIDER MANAGER
│   ├── Acquire
│   ├── Verify
│   ├── Start
│   ├── Health
│   ├── Update
│   └── Cleanup
│
└── PRIVACY DATA POLICY
    ├── Minimal Logs
    ├── No Hardware IDs
    ├── No Unnecessary Telemetry
    └── Explicit User-Controlled Diagnostics
```

---

# 54. Final Conclusions

The research changes the interpretation of the FreeIran project in an important way.

The project does not need to compete with v2rayN by having “more protocols.”

v2rayN already demonstrates that a modern client can orchestrate many cores and transports, and its current architecture contains useful ideas around TUN templates, routing, proxy groups, chains, process supervision and core management. Its current releases support a broad set of protocol/core combinations.

FreeIran's differentiating engineering direction should instead be:

```text
v2rayN-like protocol breadth
+
stronger lifecycle ownership
+
stronger evidence
+
stronger recovery
+
full-device protection
+
WFP kill switch
+
adaptive censorship detection
+
Tor/dark-network integration
+
AmneziaWG
+
central routing/DNS
+
privacy diagnostics
+
hardware-ID minimization
```

The most important conceptual shift is this:

> **FreeIran should not merely establish encrypted connections. It should continuously establish evidence that traffic is protected, detect when that protection is no longer true, and automatically change the network strategy.**

Tor and I2P extend this further by giving FreeIran additional network topologies rather than merely additional VPN protocols.

AmneziaWG adds another WireGuard-derived transport family whose traffic properties are deliberately modified for censorship resistance.

REALITY/XHTTP provide another modern censorship-resistant family through Xray.

sing-box provides the strongest existing foundation for FreeIran's full-device TUN, DNS and broad protocol surface.

Tor adds censorship-resistant access through pluggable transports and a fundamentally different trust topology, while onion services enable communication without a conventional Internet exit.

I2P provides an additional private overlay for `.i2p` services, but should remain a specialized network rather than a generic Internet VPN.

The remaining technical challenge is therefore not discovering one magic protocol.

It is engineering the complete chain:

```text
DETECT
→ SELECT
→ CONNECT
→ ROUTE
→ DNS
→ VERIFY
→ PROTECT
→ MONITOR
→ ADAPT
→ RECOVER
```

That is the architecture capable of turning FreeIran from a sophisticated proxy client into a complete autonomous connectivity and privacy system.

---

# 55. AI-Agent Handoff — Core Engineering Objective

Future coding agents should treat the following as the architectural objective:

```text
Build FreeIran into a Windows-first autonomous connectivity,
privacy and censorship-resilience engine.

Preserve the existing architecture.

Do not create duplicate:
- connection managers
- downloader systems
- core registries
- process supervisors
- routing authorities
- DNS authorities
- session models

Extend the existing foundations.

Primary goals:
1. Full-device TUN correctness
2. DNS leak prevention
3. IPv4/IPv6 leak prevention
4. WFP kill switch
5. ConnectionProof
6. Adaptive transport selection
7. REALITY/XHTTP
8. Hysteria2/TUIC/Shadowsocks
9. AmneziaWG
10. Tor + pluggable transports
11. Onion-only mode
12. I2P mode
13. Central routing engine
14. Infrastructure-aware recovery
15. Privacy diagnostics
16. Hardware-ID minimization
17. Low-metadata mode

Never represent:
Connected = Protected

Only verified evidence may produce Protected.

Never invent:
- anonymity
- privacy guarantees
- benchmarks
- network success
- censorship resistance
- hardware-ID hiding

Every feature must be:
INSPECTED
REPRODUCED
IMPLEMENTED
VERIFIED
REGRESSION-TESTED
CLEAN-ROOM VALIDATED
```

## Final engineering principle

**The model proposes. The tools execute. The laboratory verifies.**

For FreeIran, the equivalent networking principle should be:

**The transport connects. The engine decides. The evidence proves.**
