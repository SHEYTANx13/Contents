# SHEYTAN-LA Mesh Computing
## Research Paper, Architecture Decision and Implementation Blueprint

**Prepared for:** SHEYTAN-LA engineering and implementation agents  
**Author:** **Parsa Tak (SHEYTAN)**  
**Research date:** 22 November 2026  
**Repository:** [SHEYTANx13/SHEYTAN-LA](https://github.com/SHEYTANx13/SHEYTAN-LA)  
**Repository audit baseline:** `main` at `9061e0768c99918e78f656c78cecddbe13e85892`  
**Document type:** Engineering research paper and implementation blueprint  
**Status:** Proposed architecture; not an implementation or a security certification

---

## Document control and interpretation

This paper combines a source-level review of the SHEYTAN-LA repository with a technical evaluation of Go-native peer-to-peer networking, LAN discovery, internet connectivity, task scheduling, distributed execution, security and operational trade-offs.

The code audit is tied to the repository baseline above. The upstream library and standards information was checked on the research date and can change. The implementation agent must re-check exact module versions, platform support, API stability and license compatibility before adding dependencies.

Evidence labels used in this document:

- **Observed** — visible in the inspected repository or an upstream specification/documentation page.
- **Recommendation** — the proposed engineering direction for SHEYTAN-LA.
- **Validation required** — a design assumption that must be demonstrated by tests or measurements before being called implemented.

This paper does not authorize remote Git operations. Implementation must follow the repository's own agent manual: use a local non-`main` branch, preserve user changes, do not push, tag, open a pull request, merge or dispatch workflows without the operator's explicit authorization.

## Executive summary

SHEYTAN-LA should implement a **private personal compute mesh**: a user-authorized group of that user's own devices that can discover one another, exchange authenticated work, and return bounded results. The mesh should work on the local network without an internet account or external service, and should also support devices connected through different home networks, mobile networks and the public internet.

The recommended technical foundation is **Go-libp2p**, using its existing transport, identity, encrypted-channel, discovery and NAT-traversal components rather than building a custom networking protocol. The design is:

1. **LAN:** QUIC direct connections plus libp2p mDNS discovery. Add manual pairing/address exchange as a fallback for guest Wi-Fi, VLANs and networks that suppress multicast.
2. **Internet:** private rendezvous/bootstrap services for finding a user's already-paired devices; AutoNAT to estimate reachability; direct hole-punching when supported; Circuit Relay v2 when direct connectivity fails. The rendezvous and relay services should be replaceable and self-hostable.
3. **Security:** one cryptographic identity per device, explicit owner-controlled enrollment, device revocation, end-to-end authenticated encryption, application-level authorization, bounded protocols and a strict task allowlist. A libp2p peer identity proves which key connected; it does **not** grant permission to execute work.
4. **Compute topology:** one authoritative controller for a user's sessions, goals, approvals, memory and job ledger; multiple workers contribute compute. Do not begin with distributed consensus, replicated application state or leader election.
5. **Execution:** initially distribute only well-defined, bounded, low-side-effect task types. Keep approvals, arbitrary tools, persistent writes and final verification under the controller's authority. Treat remote results as untrusted input.
6. **Scheduling:** integrate with the existing Runtime Governor and measured device capabilities. Choose a worker based on compatibility, real resource headroom, model readiness, energy policy, transfer cost, network path and data locality—not simply the highest reported CPU count.
7. **Rollout:** build and prove the job model independently, then LAN networking, then internet traversal/relay, then safe Goal/multi-agent delegation, and only then broader UI and mobile-background behavior.

**Important networking truth:** internet-wide peer-to-peer computing is not achieved merely by opening a TCP/UDP listener. NAT, carrier-grade NAT, firewalls, changing addresses and mobile-network restrictions make direct inbound connectivity unreliable. A complete internet mode needs discovery/coordination infrastructure and, for some paths, a reachable relay. Direct peer-to-peer is preferred; relay fallback is part of the product design, not an exceptional afterthought. [1][2][3][4]

## 1. Product objective and requirements

### 1.1 Objective

Allow one SHEYTAN owner to make selected compute capacity from their own devices available to their other devices. Example: the owner's desktop PC can process a batch embedding or a compatible local-model inference request initiated from their laptop; the laptop can perform an indexing task for the desktop; a tablet can contribute a small supported task while it is active and permitted to do so.

The mesh is not a public volunteer-computing marketplace. It is a private, user-controlled extension of a local-first application.

### 1.2 Required properties

- **Local autonomy:** SHEYTAN-LA must remain fully functional when mesh is disabled, the internet is unavailable, rendezvous services are down or peers are offline.
- **LAN-first efficiency:** when two authorized devices share a reachable local network, discover them and use a direct route without requiring cloud coordination.
- **Internet reachability:** support authenticated device-to-device communication across networks, with transparent direct/relayed path reporting.
- **Owner control:** the owner explicitly enrolls, authorizes, pauses, removes and revokes every device.
- **Least privilege:** a worker receives only a typed job and the minimum input data needed for that job. It does not receive the owner's entire session database, memory store, tool registry or arbitrary host authority.
- **Single source of truth:** the controller remains authoritative for user-facing goals, run state, approvals, memory, final result acceptance and persistent application mutations.
- **Resource awareness:** a device can decline work because of RAM, CPU, accelerator, battery, thermal, power, storage, network or current foreground activity.
- **Failure tolerance:** peers can disconnect, sleep, change networks or crash without corrupting task state or falsely reporting completion.
- **Cross-platform honesty:** each platform is reported as supported only after native build, tests and real runtime verification for the relevant transport and background behavior.
- **Observability:** show which device executed a job, which connection route was used, what was transferred, and whether the result was accepted and verified.

### 1.3 Explicit non-goals for the first implementation

Do not start with:

- Open participation by strangers or anonymous nodes.
- A public global compute marketplace or token/economic system.
- Distributed consensus, Raft, blockchain, replicated session databases or peer leader election.
- A global DHT or public peer discovery directory.
- Distributed inference that splits a single interactive token-generation stream across arbitrary internet nodes.
- WAN tensor parallelism/model parallelism.
- Remote shell, arbitrary command execution, unrestricted browser, unrestricted filesystem access or remote tool takeover.
- Unrestricted device-to-device model weight synchronization.
- Guaranteed 24/7 workers on mobile platforms.
- A new general-purpose workflow engine that duplicates `internal/goal` or `internal/scheduler`.

These may be future research topics, but they are not prerequisites for a reliable personal mesh.

## 2. Repository architecture audit

### 2.1 Audited product architecture

The inspected `main` baseline describes SHEYTAN-LA as a local-first Go runtime. Its existing architecture follows the rule “one authority per concern.” The relevant surfaces are:

| Existing surface | Observed responsibility | Mesh integration rule |
|---|---|---|
| `internal/runtime.Stack` | Constructs and owns the process-wide runtime authorities | Construct exactly one mesh manager here; close it through the existing shutdown lifecycle |
| `internal/api` | Owns API run lifecycle, run state and presentation | Keep existing run authority; add local-only mesh status/control handlers without exposing the API to LAN/WAN |
| `internal/agent` | Executes agent runs and tools | Do not let remote peers bypass tool policy, approvals or objective verification |
| `internal/multiagent` | Planner → executor → critic → summarizer flow | Add opt-in bounded remote delegation only through an explicit typed seam |
| `internal/goal` | Durable goal state, checkpoints, bounded replanning and approval parking | Remain authoritative for goal status; remote tasks are delegated work units, not independent competing goals |
| `internal/scheduler` | Local automation schedules, durable deadline claims, cancellation and reports | Keep it as automation scheduler; do not turn it into a distributed lease ledger |
| `internal/governor` | Policy-only resource envelope and admission decisions | Extend policy inputs/decisions; it must not own transport or execute work |
| `internal/preflight` | Existing measured live resource monitor | Reuse the measurement path; never invent CPU/GPU/NPU availability |
| `internal/sandbox` | Resource-limited local code execution with platform-specific limits | Do not assume it is a universal remote-code sandbox; use typed handlers first |
| `internal/aisystem` | User-owned revisioned AI system configurations | Freeze the applicable run binding as the controller already does; transmit only task-scoped values |
| `internal/localipc` | Secured desktop IPC: named pipe on Windows, Unix socket elsewhere | Preserve unchanged; mesh is a distinct transport and trust boundary |
| `ui/fyne` | Primary native Go UI | Add mesh controls only after the core service and security contracts are tested |

Relevant code and documentation inspected include `README.md`, `docs/ARCHITECTURE.md`, `docs/ROADMAP.md`, `docs/agent.md`, `SECURITY.md`, `go.mod`, `internal/runtime/runtime.go`, `internal/scheduler/`, `internal/governor/`, `internal/goal/`, `internal/multiagent/`, `internal/agent/` and `internal/sandbox/`.

### 2.2 Critical boundary: do not expose existing endpoints

**Observed:** the current desktop app has no network listener; the desktop shell talks to the core over secured local IPC. The headless HTTP/WebSocket API binds to loopback. The Web Workspace and 3D Studio are loopback-only viewers with per-launch tokens and a read-only surface. The security policy explicitly treats weakening these boundaries as a security defect.

**Recommendation:** the mesh must be a new package and a new explicitly enabled listener/host. Do not bind the existing REST API or WebSocket endpoints to `0.0.0.0`, a LAN address or a public interface. Do not add a TCP fallback to `internal/localipc`. Mesh protocols must have their own identity, authorization and typed request surface.

This separation avoids turning the full local agent control plane into a remotely accessible service. The mesh should expose only the operations required for pairing, capability exchange, typed job dispatch, artifact transfer, cancellation and result receipt.

### 2.3 Existing compute/orchestration gap

The existing multi-agent implementation is a planner/executor/critic/summarizer flow with bounded steps and sequential hand-offs. The roadmap lists automatic sub-agent fan-out from the Goal runner as a future/near-term capability, not as a delivered distributed execution feature. The current local scheduler also has durable scheduled-task claims and cancellation, but it is not a multi-device job queue.

**Implication:** do not wire the mesh directly into arbitrary model-generated tool calls. Introduce a typed distributed-job abstraction with its own durable state and connect it to the current goal/multi-agent flow through a controlled adapter.

### 2.4 Go and platform baseline

The inspected `go.mod` declares `go 1.26.0` and `toolchain go1.27.4`. The repository builds across multiple operating-system targets, but the README is careful not to claim physical-device runtime proof where none exists. New networking dependencies must be checked against the Go version and every supported platform. The fact that a package compiles for Android or iOS does not establish that a node can run continuously in the background on those operating systems.

## 3. Networking architecture decision

### 3.1 Recommended stack: Go-libp2p

Use [`github.com/libp2p/go-libp2p`](https://github.com/libp2p/go-libp2p) as the primary Go-to-Go peer networking layer. libp2p already provides peer identities, encrypted secure channels, stream multiplexing, transport negotiation, address management and building blocks for peer discovery and NAT traversal. Its transport ecosystem includes QUIC and other choices; its connectivity components include AutoNAT, relay and hole-punching mechanisms. [1][5][6]

The product should own the **SHEYTAN application protocol, authorization, scheduling, persistence and user experience**. libp2p should own the lower-level peer transport and its well-tested protocol machinery.

Why not build a custom networking stack? A bespoke protocol would force SHEYTAN-LA to independently solve peer identity, encrypted handshakes, framing, stream multiplexing, discovery, NAT behavior, relay fallback, reconnection and resource exhaustion. These are complex security-sensitive problems and provide no unique product advantage.

### 3.2 Transport selection

| Transport | Recommendation | Purpose and trade-off |
|---|---|---|
| **QUIC** | Primary | Encrypted multiplexed streams over UDP; supports low-latency connection setup and avoids TCP application-level head-of-line blocking between independent streams. Well suited to native Go peers. [7] |
| TCP with libp2p secure channel | Fallback | Useful where UDP is blocked or heavily restricted. Must remain inside the same libp2p host and authorization model. |
| WebSocket/WebTransport | Later/optional | Consider only if browser or web-hosted peers become an explicit product requirement. Not needed for the initial native Go mesh. |
| Pion WebRTC | Not primary | A capable pure-Go WebRTC implementation with ICE/STUN/TURN and data channels, but it would introduce a second NAT/signaling/data-channel stack beside libp2p. Keep as an interoperability option if browser peers are required later. [8] |
| Raw custom UDP/TCP | Reject | Duplicates low-level transport and security responsibilities. |

QUIC is the preferred route, not a guarantee that every network allows UDP. If QUIC cannot establish a connection, the host should try a supported TCP-based route and then a configured relay. Record the actual selected route in peer status.

### 3.3 Security channel

Use libp2p's authenticated encrypted secure channels (Noise and/or TLS as provided by the selected stack configuration). QUIC itself incorporates TLS 1.3. Use the library defaults that are currently maintained unless a security review justifies a change. Do not create a second application encryption scheme without a clearly documented requirement.

Application messages still need authorization, integrity validation and scope checks. Transport encryption does not make a worker trustworthy and does not make model output safe.

## 4. Discovery and connectivity: LAN and internet

### 4.1 LAN mode

**Recommended path:** libp2p mDNS discovery followed by direct peer connection over QUIC.

mDNS uses multicast DNS records on the local network to let compatible peers discover each other without a central server. libp2p's mDNS integration can discover nodes on the same local network and provide their peer addresses. [2]

The expected flow is:

1. User enables **Local network mesh** on the selected devices.
2. Each node starts the mesh host only after the user enables it.
3. Each node advertises a minimal SHEYTAN mesh service record and its libp2p peer identity.
4. A discovered peer is shown as **unpaired / waiting for approval** until the owner completes enrollment.
5. After successful pairing, the two devices establish an authenticated direct connection.
6. They exchange versioned capability manifests and health signals; no task is dispatched until authorization and resource admission succeed.

**LAN limitations:** mDNS is scoped to multicast domains. It may fail across VLANs, guest Wi-Fi, enterprise Wi-Fi client isolation, VPN boundaries, routed subnets or multicast-filtering access points. Absence of discovery does not mean the device is offline or unsupported.

Provide two fallbacks:

- **Manual address/fingerprint pairing:** the receiving device displays a short-lived invitation and a peer address; the owner compares a fingerprint and approves the device.
- **QR invitation:** transfer a short-lived, one-use invitation containing a peer address, owner identity, enrollment nonce and requested scope. The QR is an invitation, not authorization by itself; confirmation and signature validation are still required.

LAN mode must be able to operate with no external service and no internet connection.

### 4.2 Internet mode: what is actually required

A peer behind a private NAT cannot generally accept arbitrary incoming internet connections. Dynamic public addresses, carrier-grade NAT, symmetric NAT, firewalls and UDP restrictions complicate direct peer connectivity. The IETF TURN specification explains that an intermediate relay is necessary in some cases where direct peer communication is impossible. [3]

Therefore, an internet-capable personal mesh needs three logically distinct facilities:

1. **Discovery/bootstrap:** where an online device learns how to contact the owner's other devices.
2. **NAT reachability assistance:** determine whether a peer is publicly reachable and coordinate a direct hole-punch attempt when possible.
3. **Relay fallback:** forward encrypted traffic when direct connectivity cannot be established.

These are different responsibilities. A rendezvous service helps find peers; it is not the relay that carries task data.

### 4.3 Recommended internet flow

1. An opted-in device starts its libp2p host and connects outbound to one or more configured bootstrap/rendezvous endpoints.
2. The node registers a minimal, expiring peer presence entry under an opaque owner namespace. Only the information needed to find a candidate peer is registered.
3. Another already-paired device queries the same private namespace and learns the device peer ID and usable multiaddresses.
4. The nodes authenticate each other and verify the owner-signed membership record and current revocation epoch.
5. They try a direct QUIC connection.
6. AutoNAT reachability information informs whether direct inbound connectivity is likely. When both peers and network conditions permit, libp2p's hole-punching/DCUtR machinery can coordinate a direct connection using an existing relayed path.
7. If direct connectivity fails, use a Circuit Relay v2 path from a configured relay. The relay forwards encrypted traffic; it should not receive application plaintext or job keys. [4][5]
8. The status surface reports `LAN_DIRECT`, `WAN_DIRECT`, `WAN_RELAYED`, `CONNECTING`, `OFFLINE` or a typed failure reason. Never show an opaque “connected” indicator that hides relay use.

The hole-punching option in the current Go-libp2p documentation is explicitly marked **experimental** and disabled by default. It also notes that existing long-lived streams on a relayed connection may need to be reopened after a direct connection becomes available. Treat this as an opt-in optimization with a permanent relay fallback—not a correctness dependency. [1]

### 4.4 Rendezvous topology

Use a private, user-configurable rendezvous service, preferably self-hostable. libp2p rendezvous is a discovery mechanism in which peers register and query peer presence under a namespace; the official documentation notes that the service is federated rather than fully decentralized and therefore can introduce a point of failure. [9]

Recommended design:

- The product ships with **no mandatory hard-coded public rendezvous dependency** in the initial release.
- Allow the user to select an official SHEYTAN-operated endpoint when one exists, or configure their own endpoint.
- Support more than one configured rendezvous/bootstrap endpoint where the selected protocol and library allow it.
- Use expiring records and refresh while the node is online.
- Store only opaque owner scope, peer ID, limited connection addresses, protocol compatibility, timestamp and expiry.
- Do not publish device names, filesystem paths, model file paths, user names, session IDs, prompts, job payloads or capability details beyond what is necessary to discover a peer.
- Require a separate authenticated owner-membership proof before any useful operation. A high-entropy namespace is privacy hygiene, not an authorization system.
- When the configured rendezvous service is unavailable, already connected peers and manually configured addresses should continue to work. LAN discovery must continue independently.

For a small private mesh, a global public DHT is not justified in the first implementation. Kademlia DHT introduces global routing/discovery behavior, additional resource and privacy considerations, and complexity that a private list of owner-controlled devices does not need.

### 4.5 Circuit Relay v2 topology

Provide a **SHEYTAN relay service** as an optional, replaceable infrastructure component. It may be run by the product operator or self-hosted by an advanced user. Circuit Relay v2 supports relayed peer connections and reservation/resource constraints. The relay is a transport fallback, not a trusted compute coordinator. [4]

Security and operations:

- Relay carries authenticated encrypted streams only.
- Relay must not terminate the end-to-end secure channel or process task payloads.
- Apply per-peer connection, stream, byte and reservation limits.
- Give the user a way to disable relay use or supply only private relays.
- Show relay usage, transferred bytes and connection route in the UI.
- Do not make the relay a single point of failure for LAN mode or manual direct connections.
- Make relay capacity, bandwidth, retention and data-handling policy explicit before offering a hosted relay to users.

### 4.6 Optional overlay network route

Offer a documented advanced option for users who already operate a private overlay VPN, such as Tailscale/Headscale or a WireGuard-based network. Such an overlay can make authorized devices reachable through stable private addresses, reducing the need for SHEYTAN to own public rendezvous and relay infrastructure. Tailscale documents its WireGuard-based encrypted connectivity and DERP relay fallback. [10]

This is an **optional deployment mode**, not a mandatory dependency and not a replacement for SHEYTAN's device pairing and application authorization. The application must still authenticate the peer and verify the owner membership record even when the operating system VPN makes the route private.

### 4.7 Direct vs relay truth table

| Situation | Expected behavior | Product status |
|---|---|---|
| Same LAN, multicast permitted | mDNS discover, direct QUIC | `LAN_DIRECT` |
| Same LAN, mDNS blocked | QR/manual address; direct QUIC | `LAN_DIRECT` or `MANUAL_DIRECT` |
| Different networks, public route available | Rendezvous then direct QUIC/TCP | `WAN_DIRECT` |
| NAT permits coordinated hole punch | Relay-assisted negotiation then direct connection | `WAN_DIRECT` after measured direct connection |
| NAT/firewall prevents direct path | Circuit Relay v2 | `WAN_RELAYED` |
| UDP blocked, TCP path allowed | TCP-based libp2p transport or relay | route-specific status |
| Relay/rendezvous unavailable | Retry boundedly; preserve local work; no false online state | `DEGRADED` / `OFFLINE` |
| User disables mesh | Shut down listeners and outbound mesh sessions | `DISABLED` |

Do not claim that every pair of internet connections will achieve direct P2P. The relay path is necessary for a dependable product.

## 5. Compute topology: controller and workers

### 5.1 One authoritative controller

The first version should use one controller per personal mesh:

- The controller is normally the device where the active user session or goal originated.
- It owns task planning, persistent job state, approval decisions, input scoping, result acceptance, objective verification and final user response.
- Workers advertise capabilities, accept authorized job leases, execute supported handlers and return progress/results.
- The controller can be a different device for a later run, but do not implement automatic leader election or concurrent writers to the same user database in the first release.
- A controller going offline leaves its durable queued jobs on that controller. Workers must not independently promote themselves or invent a second authoritative state.

This model is deliberately consistent with SHEYTAN-LA's existing “one authority per concern” architecture. It reduces data consistency problems, avoids a distributed consensus requirement and makes the security boundary comprehensible.

### 5.2 Role model

Each device can be configured as:

- **Controller only:** plans and dispatches work, does not accept jobs from other devices.
- **Worker only:** advertises capacity and executes allowed work from the owner-authorized controller.
- **Both:** performs both roles, subject to concurrency and resource admission.
- **Disabled:** no mesh host/listener or worker availability.

Default: **Disabled**. The user must explicitly enable the feature and separately enable worker contribution.

### 5.3 First task types to distribute

Distribute tasks with clear input/output contracts and limited side effects:

| Task family | Initial suitability | Constraints |
|---|---|---|
| Batch embeddings | High | Inputs bounded; output dimensions/model identity fixed; model readiness required |
| Independent LLM inference request | High | Send the whole request to one worker with the required model loaded; do not split one generation stream across WAN devices |
| Document/repository analysis partition | High | Read-only input snapshot; return structured findings and source references; controller reconciles |
| Independent data-analysis partitions | High | Deterministic partition contract; bounded data transfer; controller combines and verifies |
| Image/audio transformation | Medium–high | Only supported deterministic handlers; explicit file transfer and output size limits |
| Independent test/build job | Later | Use an isolated, explicit workspace snapshot; no arbitrary writeback to owner's repository; require result verification |
| General agent subtask | Later | Restrict toolset, context excerpt, deadlines, output schema and approval scope |
| Remote `shell`, unrestricted `codeExec`, browser or filesystem actions | Prohibited initially | Too much host authority and side-effect risk for an initial worker protocol |

**Do not** attempt to split each token of one interactive model generation across internet-connected devices. Autoregressive inference repeatedly depends on previous token state and can require high-bandwidth, low-latency communication. WAN coordination and model transfer can erase any compute benefit. Instead route the **whole inference request** to one peer with a compatible model already present and loaded. Model-parallel inference across devices should be a separate LAN-only research effort with measured backend support.

### 5.4 Data locality first

A task scheduler should consider where its input data already lives. Moving a large repository, dataset or model over a WAN merely to execute a short calculation can be slower and more expensive than doing the work locally. Distributed systems such as Dask explicitly document task placement that accounts for data locality and transfer cost. [11]

Rules:

- Prefer running on the device that already has the required model and source artifacts.
- Transfer compact immutable input manifests and only the referenced files/chunks needed by the selected handler.
- Do not automatically synchronize model weights as part of job dispatch.
- If the input exceeds the configured transfer budget or the estimated communication cost exceeds local execution cost, run locally or ask the user.
- Cache verified artifacts by content identity with quotas and explicit deletion policy.

## 6. Job protocol and execution correctness

### 6.1 Separate control plane and data plane

Define two versioned application protocols:

- **Control stream** — peer hello, membership proof, capability manifest, heartbeat, job offer, accept/reject, lease renewal, progress, cancellation, completion/failure and receipt acknowledgment.
- **Artifact stream** — bounded, resumable transfer of immutable input/output chunks with content identity, declared size, type and cryptographic integrity validation.

Example protocol IDs (final names are the implementation agent's decision, but they must be versioned and stable):

```text
/sheytan/mesh/control/1.0.0
/sheytan/mesh/artifact/1.0.0
```

Use generated, versioned protobuf messages (or another strongly typed schema only if the agent documents a compelling reason and compatibility plan). Avoid arbitrary unbounded JSON as a network frame format. Enforce maximum frame sizes, required fields, enum validation, schema-version checks, context deadlines and unknown-field behavior.

### 6.2 Suggested message concepts

A minimal protocol needs these message families:

| Message | Purpose |
|---|---|
| `Hello` | Exchange protocol version, peer identity, owner membership proof and supported features |
| `CapabilityManifest` | Advertise supported handlers, versions, real model identities and measured capacity bands |
| `Heartbeat` | Report liveness and current worker readiness; bounded frequency and payload |
| `JobOffer` | Propose a typed job with requirements, privacy class, input manifest, deadline and limits |
| `JobAccept` / `JobReject` | Worker explicitly accepts or rejects with typed reason |
| `LeaseGrant` / `LeaseRenew` | Establish and extend one attempt's bounded right to execute |
| `JobProgress` | Bounded phase/progress state; no unbounded log or token flood |
| `CancelJob` | Cooperatively cancel work and report settled outcome |
| `JobResult` | Return typed output manifest, completion status, worker receipt and measurements |
| `ArtifactRequest` / `ArtifactChunk` | Request and transfer bounded artifact chunks |
| `ProtocolError` | Stable machine-readable refusal/failure reason, no secret leakage |

### 6.3 Task envelope

Every dispatched job should be an immutable, versioned envelope with at least:

- Stable `job_id` for one logical operation.
- Unique `attempt_id` for a specific execution attempt.
- Monotonically increasing `lease_epoch` or fencing token.
- Owner ID and controller peer ID.
- Target worker peer ID (or explicit worker selection policy before lease creation).
- Typed handler ID and handler version; never a model-written command string.
- Envelope schema version.
- Creation time, absolute deadline and maximum run duration.
- Resource requirements and hard admission constraints.
- Privacy class and allowed data-transfer scope.
- Input artifact manifest with declared length, media/type, content digest and retention hint.
- Explicit capability set required by the task; default empty beyond the handler's intrinsic capabilities.
- Maximum result bytes and maximum progress events.
- Cancellation policy and retry classification.
- Idempotency key and owner-authenticated lease authorization.

The payload must not contain arbitrary host paths. Use opaque artifact references resolved through the worker's transfer store. Reject path traversal, symlinks escaping a job workspace, path collisions, invalid Unicode edge cases where relevant, oversized metadata and untrusted archive extraction.

### 6.4 Durable state machine

Persist every important transition before making it externally observable or sending the corresponding command.

```text
QUEUED
  -> OFFERED
  -> LEASED
  -> RUNNING
  -> RESULT_RECEIVED
  -> VERIFIED
  -> COMMITTED

Alternative terminal/parking states:
  RETRY_WAIT, PAUSED, CANCELLED, FAILED, EXPIRED, REJECTED
```

Transitions must be validated by a single ledger authority. The controller owns the authoritative job ledger. The ledger must survive application restart and must not require the network to display known jobs or their last known state.

### 6.5 Delivery and duplicate execution

Distributed execution cannot honestly guarantee that an arbitrary task body executes exactly once. A worker may finish work, lose the connection before its acknowledgment reaches the controller, and later receive another attempt. Therefore use:

- At-least-once attempt delivery with bounded retries for eligible transient failures.
- Stable `job_id`, unique `attempt_id` and incremented fencing/lease epoch.
- Idempotent worker handlers.
- Deduplicated result acceptance by job/attempt/epoch.
- Controller-side commit only when the submitted epoch is still the current authorized epoch.
- No worker-side irreversible side effects in the initial handler set.

This gives reliable, duplicate-tolerant **result commitment** for safe tasks; it is not a claim of exactly-once execution of arbitrary side effects.

### 6.6 Lease behavior

- The worker may execute only after receiving a valid current lease.
- The controller persists the lease epoch and deadline before transmission.
- Worker heartbeats or explicit lease renewals extend the deadline only after controller confirmation.
- If the controller disconnects or the lease expires, the worker cancels or parks work according to the handler contract.
- The controller may retry after a lease expires, but any late result from an old epoch is rejected.
- A peer may not unilaterally renew its own lease or alter resource/privacy limits.
- Time comparisons must tolerate clock skew; the controller's lease state and monotonic local timers are authoritative, not peer wall-clock claims.

### 6.7 Retry policy

Retry only typed transient failures such as temporary disconnect, worker shutdown, transport reset or temporary resource pressure. Use a bounded exponential backoff with jitter, maximum attempt count and maximum total job age. Do not retry malformed envelopes, authorization refusals, privacy denials, unsupported handlers or invalid result schemas. Do not use unbounded retry storms or sleep-based correctness.

## 7. Worker capability model and scheduling

### 7.1 Capability manifest

A worker reports only the fields needed to make an admission decision:

- Peer ID and membership epoch.
- Mesh protocol and handler versions.
- Operating system and architecture where relevant to a handler.
- Measured CPU capacity class and currently available worker slots.
- Available RAM where measured; unknown must remain unknown.
- Available disk quota for temporary artifacts.
- Accelerator capability only when the backend and device are verified for the required operation.
- Supported runtime/engine identity and version.
- Model IDs/hashes and readiness state; distinguish installed from loaded and loaded from actually usable.
- Battery/AC/thermal signals only where the platform can measure them reliably.
- Current load, queue depth, transfer budget and last heartbeat time.
- Handler-specific maximum input/output sizes.

Do not advertise local file paths, full hardware inventory, user account names, serial numbers or other identifying host details. Make capability disclosure minimal and opt-in.

### 7.2 Hardware truth

Reuse the project's existing truthful evidence ladder:

`detected -> available -> selected -> executed -> verified`

Do not advertise “GPU worker” just because a GPU device was detected. A capability for GPU execution requires a supported runtime/backend, a compatible loaded model or handler, and measured execution evidence. If the platform cannot measure a fact, use `unknown` and let the Governor behave conservatively.

### 7.3 Eligibility before scoring

First apply hard filters. A worker is not eligible unless:

1. It is actively paired, authorized, not revoked and protocol-compatible.
2. It is online and accepts work under the current user policy.
3. It supports the exact handler and version.
4. It satisfies model/engine constraints, if any.
5. It meets memory, CPU, disk and accelerator requirements with measured evidence.
6. It passes the Runtime Governor's admission decision.
7. The task's privacy label allows the transfer.
8. The projected data transfer is within user and system limits.
9. Its battery/AC/thermal/foreground policy permits the work.
10. Its current concurrency/queue limits have capacity.

Only eligible workers are ranked.

### 7.4 Worker score

Use a transparent policy score, not a magic “fastest device” heuristic. A conceptual cost function is:

```text
estimated_total_cost =
    queue_wait
  + worker_setup_or_model_load
  + input_transfer_time
  + estimated_compute_time
  + result_transfer_time
  + expected_retry_penalty
  + energy_or_user_preference_penalty
```

Choose the lowest acceptable expected cost subject to user preferences and fairness. Prefer a warm model and local data. Do not infer compute time from core count alone; collect benchmark measurements per handler/model/platform. Store the score components so the UI can explain “why this device was selected.”

### 7.5 Owner controls

Expose user-configurable limits such as:

- Enable/disable mesh globally, LAN mode and internet mode separately.
- Allow this device to contribute CPU, GPU or inference only when actually supported.
- Contribute only while plugged in / not in battery saver.
- Maximum worker CPU share, memory headroom and concurrent jobs.
- Maximum upload/download bandwidth and per-job transfer size.
- Maximum worker runtime and maximum result size.
- Allowed task families and privacy classes.
- Whether background execution is permitted on the platform.
- Whether the device may use an external relay or only user-operated relays.

Default to conservative limits. The user must be able to pause all worker activity immediately.

## 8. Security, privacy and trust model

### 8.1 Threat model

Assume any enrolled device may become compromised, run an old client, contain buggy code, be stolen, or return malformed or misleading results. Assume the local network may contain untrusted devices. Assume rendezvous and relay operators can observe connection metadata and traffic volume, even if payloads remain end-to-end encrypted. Treat every received message, file and compute result as untrusted until validated.

A “personal mesh” lowers the expected adversary population but does not eliminate device compromise or software vulnerabilities.

### 8.2 Device identity and enrollment

- Give every device a separate persistent cryptographic keypair and libp2p peer identity.
- Do not copy the controller's root private key to every device.
- Keep the owner/root signing key separate from ordinary device identities. Use a platform secure keystore where available; investigate current repository secrets/key handling before choosing storage. Never put private keys in `config.json`, logs, crash dumps or export bundles.
- Create one-time, short-expiry enrollment invitations with a cryptographically random nonce, owner identity, intended scope and request expiry.
- Require explicit approval on an already-authorized owner device and fingerprint confirmation on the new device.
- Issue a signed membership record binding device peer ID, owner ID, role, allowed task classes, issue/expiry time and membership epoch.
- Removal/revocation increments the owner's membership epoch and invalidates the device on the next authenticated contact.
- Use short-lived work leases and re-check membership for every new job. A disconnected/revoked device cannot receive an immediate remote revocation signal while offline; short-lived authorization and refusal on reconnect limit that exposure.

### 8.3 Authentication is not authorization

libp2p peer IDs authenticate the cryptographic identity of a peer, but the application must implement authorization mapping that peer to an owner membership and allowed actions. The libp2p security documentation explicitly cautions that application-level access control remains the application developer's responsibility. [5]

Every incoming request must check:

- Secure transport identity matches the claimed peer.
- Owner membership signature is valid and current.
- The membership epoch is current.
- Role permits the operation.
- Requested handler is allowlisted for that peer.
- Job lease is current and signed by the controller.
- Resource/data limits are within policy.

Never trust a peer because it is on the LAN, shares a VPN address, has a matching device name, or possesses an unverified invite string.

### 8.4 Privacy classes

Start with three privacy classes and fail closed:

| Class | Meaning | Default behavior |
|---|---|---|
| `LOCAL_ONLY` | User data must not leave its source device | Never dispatch remotely |
| `OWNER_MESH` | May be sent only to explicitly enrolled devices belonging to this owner | Require paired peer, matching membership and supported handler |
| `EXPLICIT_SHARE` | User has explicitly approved this specific data/job for remote transfer | Require a durable, job-scoped user approval record |

Do not create a vague global “share my data” setting that silently exposes every conversation, document or repository. Privacy scope should be attached to the task and input manifest.

### 8.5 Execution containment

Initial remote workers should execute **compiled-in, typed handlers**. The worker's handler registry should map known handler IDs to Go implementations with explicit input/output schema, quotas, timeout and capability requirements.

Do not remotely expose:

- Arbitrary shell commands.
- General `codeExec` with unrestricted code.
- Local browser control.
- General filesystem access.
- Credential stores, application settings or user profile directories.
- The controller's approvals or secrets.

If arbitrary cross-platform user code becomes a requirement, evaluate a separate WASM module execution path with a restricted host import set and resource limits. `wazero` is a pure-Go WebAssembly runtime without platform dependencies, but a WASM sandbox still depends on carefully controlling host imports and must not be described as a complete OS-level security boundary. [12]

The current `internal/sandbox` documentation describes differing platform enforcement: Windows Job Objects, best-effort Linux resource limits and timeout-only behavior on macOS. Do not overstate those controls as an equivalent universal sandbox.

### 8.6 Artifact handling

- Use an opaque job-scoped workspace.
- Enforce declared byte limits before allocating or writing data.
- Validate content digest, declared length, file type and schema.
- Reject unexpected files and archive traversal.
- Do not resolve user-provided arbitrary paths on the worker.
- Do not allow output to overwrite an application path directly.
- Make result artifacts immutable; controller imports them through its existing typed stores and approval rules.
- Use retention quotas and cleanup after terminal jobs, while retaining only user-requested outputs.

### 8.7 Denial-of-service and resource limits

Set explicit libp2p connection/resource-manager limits; limit peers, connections per peer, streams, pending handshakes, message sizes, artifact sizes, transfer rates, task concurrency, per-peer queue depth and total worker RAM. The go-libp2p API documents the ResourceManager and recommends configuring resource limits; its DoS guidance should be part of the implementation review. [1][5]

Also require:

- Strict connection and stream deadlines.
- Bounded backpressure and bounded progress queues.
- Maximum heartbeat rates.
- Per-peer token/rate buckets.
- Safe parsing and fuzz tests for all network message families.
- No automatic decompression without size and expansion-ratio bounds.
- Cooperative cancellation and bounded shutdown.
- No network-controlled process termination, engine restart or system settings mutation.

### 8.8 Logs and privacy telemetry

Record stable peer IDs or privacy-safe aliases, task IDs, handler names, timestamps, byte counts, route type, state transitions, failure codes and measured durations. Do not log prompts, full input payloads, output bodies, credentials, invitations, tokens, content hashes of sensitive documents if those become identifying, private keys or raw peer metadata beyond necessity. Sanitize peer-controlled strings and cap all log fields.

## 9. Integration blueprint for SHEYTAN-LA

### 9.1 New package boundary

Create a single new production authority, initially under `internal/mesh/`. Do not spread transport ownership across UI/API/agent packages.

Suggested initial layout (adapt after inspecting the current file catalog and naming conventions):

```text
internal/mesh/
  manager.go             # exactly-one manager lifecycle; Start/Close/status
  config.go              # validated, non-secret settings and feature flags
  identity.go            # per-device identity and owner enrollment records
  membership.go          # pairing, authorization, revocation and epochs
  host.go                # libp2p host construction and resource limits
  discovery_lan.go       # mDNS advertisement and discovery
  discovery_wan.go       # rendezvous/bootstrap registration and refresh
  connectivity.go        # direct/relay classification and reconnect state
  protocol.go            # protocol IDs, version negotiation, stream bounds
  capability.go          # measured handler/model/device manifest
  job.go                 # immutable job/envelope types and validation
  ledger.go              # durable job state, attempt IDs and fencing epochs
  coordinator.go         # offers, lease grant/renewal, retry and commit
  worker.go              # allowlisted typed handler registry and execution
  scheduler.go           # eligibility filters and transparent worker scoring
  artifact.go            # bounded content-addressed transfer and verification
  metrics.go             # sanitized mesh observations
  *_test.go              # deterministic tests, fake peers and integration tests
  protocol/v1/mesh.proto  # versioned wire schema, if protobuf is selected
```

The first version may keep some of these files smaller or combine adjacent files. The important rule is one mesh manager and clearly separated responsibilities—not a large package explosion.

### 9.2 Runtime lifecycle integration

`internal/runtime.Stack` should construct exactly one `*mesh.Manager` when mesh capability is compiled in, but its host/listeners remain stopped while mesh is disabled. Construction should not bind sockets or begin discovery as a hidden side effect.

Provide explicit lifecycle operations such as:

```go
Start(ctx context.Context) error
Stop(ctx context.Context) error
Close(ctx context.Context) error
Status() Status
```

The concrete API may differ. It must support idempotent start/stop, context cancellation, one owner for lifecycle state and bounded shutdown. `Stack.Close` should stop accepting new mesh work, cancel or settle active worker tasks, close peer streams/host, flush the ledger and return joined shutdown errors in the same manner as existing authorities.

### 9.3 API integration

Add a focused `internal/api/mesh.go` surface for local UI status and user-authorized operations. It must be served only by the existing loopback API and must not expose peer control via the old `localipc` or general remote HTTP listener.

Suggested endpoints (names are illustrative):

```text
GET    /api/mesh/status
GET    /api/mesh/peers
GET    /api/mesh/jobs
POST   /api/mesh/enable
POST   /api/mesh/disable
POST   /api/mesh/pair/invite
POST   /api/mesh/pair/approve
POST   /api/mesh/peers/{id}/revoke
POST   /api/mesh/jobs/{id}/cancel
```

All mutating routes must use the existing API's validation and approval conventions. Do not allow arbitrary network-supplied path IDs to bypass owner authorization. Return typed errors and redact invitations/secrets.

### 9.4 Governor integration

The mesh manager must not implement a competing CPU/RAM sampler. Consume the existing Governor envelope and measured `preflight.LiveMonitor` data. Add only the policy facts needed for remote work, for example worker CPU budget, memory headroom, active inference pressure, background concurrency and whether a new transfer is allowed. The Governor recommends/admission-checks; the mesh worker and coordinator enforce at their existing control points.

Remote task admission should be fail-closed if a required resource fact is unknown. A device under critical pressure must cooperatively cancel/pause background worker tasks according to handler policy, without process killing.

### 9.5 Goal and multi-agent integration

First deliver the mesh as a standalone typed compute service with deterministic test tasks. Only then add remote delegation to `internal/goal` and `internal/multiagent`.

The integration should look like:

```text
Goal / multi-agent planner
        |
        v
Typed subtask request
        |
        v
Controller policy + privacy + approval + Governor checks
        |
        v
Mesh coordinator -> selected authorized worker
        |
        v
Worker's fixed handler registry
        |
        v
Result manifest + worker receipt
        |
        v
Controller validates, records evidence and verifies
        |
        v
Existing Goal / run state continues
```

The model must not be allowed to invent peer IDs, set lease epochs, widen privacy labels, invoke handlers outside the allowlist or command a remote shell. The planner proposes a typed subtask; deterministic application policy decides whether and where to execute it.

### 9.6 Scheduler boundary

Keep `internal/scheduler` for local scheduled automation and event triggers. Create a separate mesh job ledger/lease manager for distributed execution. The mesh may be invoked by the scheduler through an injected interface later, but the scheduler must not become the owner of peer transport or remote leases.

### 9.7 UI integration

Use the existing Fyne native shell. Add a Mesh destination or a compact System Centre panel only after pairing, connectivity, job and security tests pass. UI must display:

- Mesh status and whether mesh is disabled, LAN-only or internet-enabled.
- Role (controller/worker/both).
- Device list and pairing/revocation state.
- Device online status, last seen and connection path (`LAN_DIRECT`, `WAN_DIRECT`, `WAN_RELAYED`).
- Supported handlers and actual model readiness.
- Worker resource limits and current utilization.
- Active/queued/failed jobs and progress.
- Per-job data-transfer size and destination device.
- Pause-all, pause-worker, disconnect, forget and revoke operations.

A peer should never appear as trusted merely because it was discovered. Clearly distinguish discovered, paired, authenticated, authorized and currently connected.

## 10. Dependency and alternative evaluation

### 10.1 Candidate comparison

| Candidate | Strengths | Weaknesses for SHEYTAN-LA | Decision |
|---|---|---|---|
| **Go-libp2p** | Go-native; peer identity; secure channels; multiplexed protocols; QUIC; mDNS; rendezvous; AutoNAT; hole punching; Circuit Relay | Requires a careful app-level membership/authorization design; internet deployment needs bootstrap/rendezvous and relay operations; hole punching is not guaranteed | **Primary recommendation** |
| Pion WebRTC | Mature WebRTC API, ICE, STUN/TURN, data channels, pure Go | Requires signaling and its own peer model; overlaps networking responsibilities with libp2p; browser interoperability may be unnecessary | Future browser interoperability only |
| Tailscale / Headscale / WireGuard | Convenient private network reachability; strong overlay tunnel; can simplify internet routes | User must install/configure or operate overlay infrastructure; not a compute scheduler; still needs SHEYTAN application pairing and authorization | Optional advanced route |
| Hashicorp memberlist | Gossip membership and failure detection | Does not itself solve encrypted application protocols, NAT traversal or relay; another membership authority | Do not use as core |
| NATS JetStream | Durable messaging and replay/at-least-once delivery | Requires a NATS server; introduces external infrastructure and another job/message authority | Not needed initially |
| Temporal Go SDK | Durable workflow/activity orchestration | Requires Temporal service; duplicates existing Goal/Scheduler authorities for this personal mesh | Not needed initially |
| Kademlia DHT / Gossipsub | Useful for global peer routing or large topic-based overlays | Global discovery/pubsub is excessive and increases privacy/operational surface for one user's small device set | Defer |
| Custom TCP/UDP mesh | Full control over wire format and connection behavior | Reimplements security, peer discovery, framing, NAT, reconnection and abuse protection | Reject |

### 10.2 Library version and compatibility policy

At research time, pkg.go.dev surfaced `go-libp2p` v0.52.0 as published on 14 November 2026, while the GitHub Releases page captured during this review still listed v0.50.0 as its latest visible release. Treat this as a version metadata discrepancy and resolve it during implementation. Do not select a dependency from a search result alone.

Before pinning:

1. Inspect the exact module tag and upstream `go.mod`.
2. Confirm module proxy availability and checksum.
3. Confirm Go 1.26 language/toolchain compatibility with the repository's toolchain.
4. Check transitive dependencies, platform files, CGO requirements and security advisories.
5. Build and test Windows, Linux, macOS and supported mobile targets that the selected library claims to support.
6. Run real LAN and WAN/relay tests; cross-compilation alone is not runtime proof.
7. Pin an exact stable version and commit the corresponding `go.sum` changes.

Do not use an unstable pseudo-version just to obtain a hole-punch feature. Keep experimental NAT traversal behind a configuration/feature gate.

### 10.3 Dependency minimization

Initially add only the required Go-libp2p module and any strictly necessary protocol code-generation tool. Do not simultaneously introduce Pion, NATS, Temporal, memberlist, a second scheduler or a new database unless evidence demonstrates a concrete requirement. Reuse existing logging, configuration, Governor, storage and test patterns.

## 11. Implementation phases and exit gates

Implementation should be delivered as small, independently verifiable phases. A feature is not complete because documentation, compilation or a single happy-path test exists.

### Phase 0 — Repository preflight and architecture freeze

**Objective:** establish the exact current branch/HEAD, runtime wiring and dependency compatibility before changing production code.

Required work:

- Read the user's work order, `docs/agent.md`, `docs/ai-notes.md`, `docs/ROADMAP.md`, `docs/ARCHITECTURE.md`, `SECURITY.md`, `docs/CONTRIBUTING.md`, `README.md`, current `go.mod` and relevant package tests.
- Inspect live `git status`, `git log`, branch list and current HEAD. Preserve all dirty/untracked work.
- Create a local non-main working branch from the freshly inspected HEAD.
- Map existing key management, config persistence, runtime start/close order, local API routes, resource manager and platform test lanes.
- Prototype Go-libp2p host construction and a fake two-peer test without starting a production listener.
- Produce a short architecture decision record covering chosen module version, license, Go version, transports, identity storage, test feasibility and known platform gaps.

**Exit gate:** no production networking enabled; dependency compatibility and lifecycle design reviewed; unit-test strategy and platform matrix documented.

### Phase 1 — Typed job model and durable ledger (no network)

**Objective:** prove correctness of distributed work semantics before connecting devices.

Required work:

- Define immutable `JobEnvelope`, `JobAttempt`, `Lease`, `CapabilityManifest`, `ArtifactManifest` and `JobResult` structures.
- Implement strict validation, typed failure codes and versioning.
- Implement controller-owned durable job state transitions and append/commit semantics using the repository's existing persistence conventions.
- Implement unique attempt IDs, monotonic fencing epochs, expiration, cancellation and stale-result rejection.
- Build a deterministic fake-worker interface and tests for restart recovery, duplicate delivery, worker timeout, late result, lease expiration and cancellation.
- Add bounded retry policy for explicitly transient error categories only.

**Exit gate:** the fake-worker test suite proves stale and duplicate results cannot corrupt the ledger; restart recovery restores truthful job state; `go test -race` passes for touched packages.

### Phase 2 — Identity, pairing and authorization

**Objective:** build the device trust boundary before exposing useful job capabilities.

Required work:

- Generate and persist one keypair per device using the app's secret/keystore conventions.
- Implement one-time invitation, expiry, owner approval, fingerprint confirmation, signed membership, role/scope validation and revocation epoch.
- Implement negative tests: unpaired peer, expired invitation, replayed invitation, tampered membership, wrong owner, revoked peer, wrong role, protocol downgrade and unauthorized handler request.
- Confirm that discovery alone never changes peer state to paired or authorized.

**Exit gate:** all security-negative tests fail closed; no key, invitation secret or authorization token enters logs or config exports.

### Phase 3 — LAN networking

**Objective:** two or more paired devices can run bounded typed tasks over a local network.

Required work:

- Add exactly one libp2p host manager with QUIC and necessary TCP fallback.
- Add mDNS discovery behind the mesh setting.
- Add manual/QR address fallback for networks that block multicast.
- Add version negotiation, bounded control/artifact streams and resource-manager limits.
- Add capability exchange, worker accept/reject, lease flow, cancellation, artifact transfer and result acceptance.
- Test on actual devices in one LAN; also test multicast disabled/client isolation and manual connection fallback.

**Exit gate:** verified physical 2-node and 3-node LAN task execution; no existing API/IPC exposure changes; route status truthful; mesh disabled means no mesh listener/discovery.

### Phase 4 — Internet discovery, NAT traversal and relay

**Objective:** authorized devices on separate networks can connect and execute a bounded job.

Required work:

- Deploy/configure a private test rendezvous/bootstrap service.
- Register and refresh expiring device records; implement discovery independent of LAN mDNS.
- Add AutoNAT reachability observation and direct connection attempts.
- Add relay v2 fallback and route-state observability.
- Experiment with libp2p hole punching only behind an explicit feature gate; it must not block relay fallback.
- Test at least two realistic NAT/network combinations, including a path where direct connections fail and relay works.
- Verify relay cannot decrypt application payloads and does not receive job data outside encrypted streams.
- Document optional overlay VPN configuration.

**Exit gate:** two different networks complete a real authenticated task via direct or relayed path; tests prove relay fallback and typed behavior on rendezvous outage; network path is visible; no false claim that all NATs are directly traversable.

### Phase 5 — Safe compute handlers and Governor admission

**Objective:** turn peer connectivity into useful, safe compute capacity.

Required work:

- Implement a small allowlist of pure/bounded handlers, starting with batch inference or embeddings only when supported by existing engine/model readiness; add deterministic data/analysis handlers as suitable.
- Keep arbitrary code, shell, browser and general file access unavailable.
- Integrate worker admission with existing Governor and measured hardware facts.
- Implement task scoring with data locality, model warmth, transfer estimates, resource pressure, energy policy and user preference.
- Record per-handler benchmarks; do not infer speed from hardware descriptions alone.
- Add per-job input/output budgets, concurrency caps and worker idle/foreground behavior.

**Exit gate:** tests and measured runs show that eligible remote jobs execute only on the intended peer; local fallback works; a peer under pressure rejects/pauses safely; performance claims include reproducible measurements.

### Phase 6 — Goal and multi-agent delegation

**Objective:** permit selected Goal/multi-agent subtasks to use the mesh without moving authority.

Required work:

- Add an injected typed `RemoteExecutor`/delegation seam to the current Goal or multi-agent authority, not a second planner.
- Default remote delegation off until user enables it.
- Bound fan-out, context excerpt size, worker count, timeouts, artifact volume and replan budget.
- Apply privacy, authorization, approval and Governor checks before sending any task.
- Feed worker result receipts and controller verification evidence into the existing Goal/run record.
- Ensure a failed remote task can be retried locally when safe, without duplicating side effects.

**Exit gate:** end-to-end Goal integration tests show correct checkpoint, approval, cancellation, restart and final verification behavior; worker output alone cannot mark a Goal complete.

### Phase 7 — User experience, platform rollout and operations

**Objective:** expose a clear and manageable product feature.

Required work:

- Add Fyne controls for role, opt-in, pairing, worker policies, peer status, jobs, route, data transferred and revocation.
- Add diagnostics that expose typed network errors and sanitized peer/job metadata.
- Document LAN-only mode, internet mode, relay cost/privacy, battery policy, limitations and recovery.
- Validate Windows/Linux/macOS runtime, then Android foreground worker behavior; assess iOS restrictions separately.
- Test upgrade/downgrade and incompatible protocol versions.
- Add security scanning, fuzzing, race tests and dependency review to the existing CI model.

**Exit gate:** end-user can pair, pause, remove and revoke a device; mesh can be disabled cleanly; cross-platform status is based on actual platform evidence; documentation matches behavior.

## 12. Verification matrix

| Area | Required tests/evidence | Pass condition |
|---|---|---|
| Job state | Unit/property tests for all transitions | Invalid transitions rejected; restart recovers durable state |
| Idempotency | Duplicate offer, duplicate result, stale epoch, late result | No stale/duplicate result is committed twice |
| Pairing | valid/invalid/replayed/expired invitation and owner mismatch | Only the authorized owner can enroll a peer |
| Revocation | revoke online peer; reconnect revoked peer; stale membership | New jobs denied; offline revocation limitation documented; no stale lease accepted |
| Protocol | malformed frames, unknown version, oversized field, truncated stream | Bounded failure; no panic, leak or unbounded allocation |
| LAN discovery | same subnet, mDNS blocked, manual address, multicast noise | Paired peers discover/connect; unpaired nodes stay unauthorized |
| Direct WAN | reachable public peer and supported NAT test lab | Direct route verified and surfaced accurately |
| Hole punching | symmetric/CGNAT/UDP-blocked variants | Best-effort success where feasible; relay fallback remains reliable |
| Relay | different NATs, relay saturation, relay outage | Relay works under limits; failure is typed; payload remains end-to-end encrypted |
| Data transfer | truncation, digest mismatch, disk full, large chunk, resume | Partial data never becomes a committed artifact; limits enforced |
| Worker admission | low RAM, CPU pressure, engine busy, battery policy, unknown metrics | Worker rejects or defers safely; no fabricated resource facts |
| Compute handler | deterministic fixtures, invalid input, timeout, cancellation | Handler returns bounded typed result or typed error |
| Local fallback | peer disappears during work | Local execution or durable retry is safe; stale remote completion cannot commit |
| Runtime lifecycle | start twice, stop twice, shutdown during handshake/job/transfer | No goroutine leak; bounded close; no new work after shutdown begins |
| Security | fuzz parsers, permission matrix, race tests, dependency scan | No bypass, panic or uncontrolled resource growth in tested surfaces |
| Existing product | all existing tests and IPC/API contracts | Mesh changes do not weaken local IPC, loopback-only APIs or current behavior |
| Platforms | native build, test and runtime by OS/architecture | Status distinguishes build, test, CI and physical-device evidence |

Suggested Go verification commands (adapt to repository scripts and platform prerequisites):

```bash
gofmt -w internal/mesh internal/runtime internal/api
go test ./internal/mesh/... -count=1
go test -race ./internal/mesh/... -count=1
go test ./internal/runtime/... ./internal/governor/... ./internal/goal/... ./internal/multiagent/... -count=1
go vet ./internal/mesh/... ./internal/runtime/... ./internal/api/...
go test ./...
```

Also run the repository's documented full CI gates. If the host lacks Fyne/CGO prerequisites, use the project's supported headless checks and report the missing native gate as **NOT RUN**. `go build` is not proof that networking works at runtime. Use real two-device tests and record which network route was used.

Network protocol parsers and artifact handling should be fuzzed. Race detection should cover host shutdown, peer disconnect, lease expiry, cancellation, concurrent job offers and result commit. Go's security guidance recommends vulnerability scanning and fuzz testing as part of the development process. [13]

## 13. Performance measurement plan

Do not claim “uses all devices” or “faster” without measuring workload-specific improvement. Define baseline and mesh measurements on the same hardware and data.

Capture:

- Local execution duration vs remote execution duration.
- Queue wait, connection setup, handler setup/model load, input transfer, compute and result transfer separately.
- First-token/total inference latency for whole-request remote inference.
- Effective bytes/sec, RTT, packet loss and relay/direct path.
- Worker CPU/RAM/accelerator use and controller resource use.
- Energy/battery cost where measurable.
- Task success rate, retry rate, cancellation latency and worker availability.
- Data locality and warm model effects.

Test at least:

1. Same task on the controller alone.
2. Same task on one LAN worker with warm model.
3. Same task on one LAN worker with cold model.
4. Same task over a WAN direct route.
5. Same task over a relayed route.
6. A task whose input is deliberately too large to benefit from offload.

The scheduler should prefer local execution when the expected remote cost is worse. A valid mesh is allowed to decide **not** to distribute a task.

## 14. Operational model and deployment choices

### 14.1 Minimum viable deployment

- No external infrastructure for LAN-only mode.
- For internet mode, a configured reachable rendezvous/bootstrap endpoint and at least one reachable relay are needed for the full fallback experience unless the user supplies an overlay VPN or manually reachable peer route.
- The service operator should expose a minimal discovery endpoint and a relay service with separately documented retention, bandwidth and abuse controls.
- Users should be able to run their own endpoints and remove the hosted defaults.

### 14.2 Hosted infrastructure policy

If SHEYTAN later operates rendezvous and relay infrastructure:

- Publish data flow and metadata retained.
- Store only expiring connectivity metadata; never store job plaintext or model prompts.
- Keep relay end-to-end encrypted and avoid payload logging.
- Offer capacity/bandwidth limits and abuse prevention that do not require inspecting task contents.
- Provide a self-hosted alternative and a way to disable hosted endpoints.
- Make service availability and bandwidth usage visible; do not claim the system is fully serverless if it depends on an operator relay.

### 14.3 Mobile lifecycle

Mobile OS background-execution restrictions mean an Android/iOS device may not remain an always-on worker. Treat mobile devices as opportunistic workers while the app is foregrounded or while the OS explicitly grants permitted background runtime. The controller must detect disconnects and expire leases normally. Do not use hidden keep-alive hacks, excessive wake locks or misleading “online” states.

## 15. Key risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| NAT traversal fails | Internet peer unreachable | Direct attempts + relay fallback; explicit route state; overlay/manual option |
| Rendezvous outage | New discovery fails | Multiple/configurable endpoints; existing connections and LAN discovery continue |
| Compromised worker | Data exposure or forged output | Minimal data scope, typed handlers, worker untrusted, controller verification, revoke device |
| Malicious LAN node | Unauthorized task or protocol attack | Pairing required, authenticated membership, no trust by LAN presence |
| Duplicate attempts | Duplicate work or side effects | Idempotent handlers, job/attempt IDs, fencing epochs, no early arbitrary side effects |
| Large transfers | Slow work and bandwidth use | Data locality, transfer thresholds, quotas, resumable bounded chunks |
| Resource contention | Host freeze or inference slowdown | Governor admission, worker limits, cooperatively pause background work |
| Model mismatch | Wrong or unsupported execution | Exact model/engine identity and handler version checks |
| Platform variation | Broken networking on target OS | Native platform matrix and runtime tests; honest status labels |
| Dependency drift | API or security breakage | Exact pin, module verification, security review, controlled upgrades |
| Scope expansion | Mesh destabilizes core app | Phased rollout, one authority, no broad tool delegation in v1 |

## 16. Implementation agent work order

The following is the operative implementation instruction. The agent must treat this paper as the design baseline, but must still verify all referenced source files against the current repository HEAD before editing.

### Mission

Implement the first production-quality increment of SHEYTAN-LA's private personal compute mesh in Go, following this paper and the current repository contracts. Deliver in independently verifiable phases; do not attempt all phases as one unreviewable rewrite. The first implementation goal is a secure typed job model and deterministic local/fake-worker execution, followed by LAN connectivity. Internet rendezvous and relay are subsequent gated milestones.

### Non-negotiables

1. **Never push, tag, dispatch workflows, open a PR, merge or write to `main`.** Work on a local non-main branch only. Remote operations require a separate explicit operator authorization.
2. Read the current `docs/agent.md`, `docs/ai-notes.md`, `docs/ROADMAP.md`, `docs/ARCHITECTURE.md`, `SECURITY.md`, `docs/CONTRIBUTING.md`, `README.md`, `go.mod`, current code and tests before editing.
3. Run `git status`, `git log --oneline -5`, `git branch -a`, inspect HEAD and record dirty/untracked files. Preserve every pre-existing user change.
4. Do not weaken the existing `internal/localipc` boundary, Windows named-pipe DACL/SID checks, Unix peer-credential checks, loopback-only HTTP/WS APIs, per-launch viewer tokens or security headers.
5. Mesh is disabled by default. No listener, discovery, relay, rendezvous or network worker starts as a side effect of app construction.
6. No unpaired peer may execute any handler. Discovery is not trust. libp2p authentication is not application authorization.
7. Do not expose arbitrary shell, `codeExec`, filesystem, browser or credential access to a remote worker in the first release.
8. Keep controller-owned durable state. Do not add consensus, replicated sessions, leader election, a second scheduler or a second goal engine.
9. Reuse the current Governor and measured resource monitor. Unknown measurements remain unknown and are handled conservatively.
10. Do not claim cross-platform runtime support from a successful cross-compile. Keep build, test, CI and physical-device evidence separate.
11. No sleep-based correctness, retry storms, forced process killing, fake telemetry, weakened tests, hidden background keep-alives or false performance claims.
12. Update affected docs in the same change. Keep `docs/ai-notes.md` a current factual hand-off, not an append-only diary. Do not edit human-only `docs/Dev-Notes.md` unless separately authorized.

### Required phase-one deliverables

- Architecture decision record for selected libp2p module version and connectivity plan.
- `internal/mesh` job types, validation, durable ledger and typed failure codes.
- Fake worker interface and deterministic end-to-end tests.
- Attempt identity, lease expiry, fencing epoch, stale-result rejection, cancellation, retry cap and restart recovery.
- No production network listener enabled yet.
- A documented file-by-file mapping to `runtime.Stack`, Governor, Goal and multi-agent integration points for the next phase.

### Required phase-two deliverables

Only after phase one is green:

- Per-device identity and secure enrollment/revocation.
- libp2p host lifecycle with mDNS and direct LAN transport.
- Pairing and authorization tests before enabling any compute handlers.
- A two-device LAN demonstration with a fixed, harmless typed task.
- Confirm mesh off means no listener/discovery and existing API/local IPC contracts remain unchanged.

### Gates and verification

For each phase:

1. Explain the exact root problem and intended behavior before editing.
2. Identify exact files and tests to add/change.
3. Implement the smallest complete vertical slice.
4. Run `gofmt`, focused tests, race tests for concurrency surfaces, `go vet`, supported build gates and relevant existing contract tests.
5. Run eight focused recheck/enhancement cycles on the delivered slice: protocol/contract review; security/authorization review; persistence/recovery review; concurrency/race review; resource/backpressure review; cross-platform review; documentation/compatibility review; final regression review. Each cycle must inspect actual code/tests and record concrete findings or state “no defect found”; do not manufacture changes to satisfy a count.
6. Re-run all affected tests after the final cycle.
7. State what was locally tested, what was built only, what ran on hosted CI, what was physically tested and what remains NOT RUN.
8. Produce a concise implementation note and exact next step. Keep deliverables in the agent's requested download location when that environment provides one; do not fabricate artifacts or evidence.

### Definition of done

A phase is done only when its requirements are implemented, tests prove both positive and negative behavior, docs match the code, all applicable gates are green, and unrun platform/network tests are stated honestly. Do not label the entire mesh as complete after LAN support alone; LAN and internet modes have separate exit gates.

## 17. Final architecture decision record

**Decision:** Adopt Go-libp2p for the SHEYTAN-LA private personal device mesh. Use QUIC as the preferred native transport; mDNS for LAN discovery; manually approved peer pairing; a private configurable rendezvous/bootstrap service for internet discovery; AutoNAT and best-effort hole punching for direct reachability; Circuit Relay v2 as a required internet fallback; and an optional overlay VPN route for advanced users.

**Topology:** one authoritative controller with multiple owner-approved workers. The controller owns job ledger, goals, sessions, approvals, memory, result verification and persistent writes. Workers execute typed allowlisted handlers under short-lived leases and resource limits.

**First distributed workloads:** whole-request inference on one model-ready peer, embeddings and bounded deterministic read-only batch tasks. No WAN tensor-parallel inference and no arbitrary remote tools in the first release.

**Security posture:** feature off by default; per-device keys; signed owner membership; explicit pairing; peer revocation; application authorization; end-to-end encrypted streams; bounded protocols; strict capability and data scope; worker output untrusted; controller verification mandatory.

**Deliberately deferred:** global DHT, pub/sub, NATS, Temporal, memberlist, public compute marketplace, arbitrary code distribution, decentralized consensus and general cross-device model parallelism.

**Why this decision:** it solves the LAN and internet connectivity problem without compromising SHEYTAN-LA's local-first design or its single-authority runtime. It reuses a maintained Go-native networking ecosystem and preserves an incremental path from safe local/fake jobs to LAN execution to internet relays and controlled Goal delegation.

---

## References and source material

All upstream links below were checked or consulted on 22 November 2026 unless the page itself provides a publication date.

1. **Go-libp2p package documentation, v0.52.0** — API options for AutoNAT v2, relay, AutoRelay, resource manager and experimental hole punching. The package page lists publication on 14 November 2026; confirm tag provenance/version metadata before pinning.  
   https://pkg.go.dev/github.com/libp2p/go-libp2p
2. **libp2p mDNS documentation** — local-network peer discovery via multicast DNS.  
   https://libp2p.io/docs/mdns/
3. **IETF RFC 8656 — Traversal Using Relays around NAT (TURN)** — explains cases where direct peer communication is impossible and an intermediate relay is needed.  
   https://www.rfc-editor.org/rfc/rfc8656.html
4. **libp2p Circuit Relay documentation** — relay path behavior, reservations and resource constraints.  
   https://libp2p.io/docs/circuit-relay/
5. **libp2p Security Considerations and DoS Mitigation** — peer identity, application authorization responsibilities, connection/resource limits and abuse considerations.  
   https://libp2p.io/docs/security-considerations/  
   https://libp2p.io/docs/dos-mitigation/
6. **libp2p project and specifications** — modular P2P architecture and protocol specifications.  
   https://libp2p.io/  
   https://github.com/libp2p/specs
7. **libp2p QUIC documentation and IETF RFC 9000** — QUIC transport characteristics and protocol standard.  
   https://libp2p.io/docs/quic/  
   https://www.rfc-editor.org/rfc/rfc9000.html
8. **Pion WebRTC** — alternative pure-Go WebRTC implementation for possible future browser interoperability.  
   https://github.com/pion/webrtc
9. **libp2p Rendezvous documentation** — namespace-based peer registration/query model and operational trade-offs of rendezvous services.  
   https://libp2p.io/docs/rendezvous/
10. **Tailscale documentation: WireGuard and DERP** — optional overlay network and relay model.  
    https://tailscale.com/docs/concepts/wireguard  
    https://tailscale.com/docs/reference/derp-servers
11. **Dask Distributed: Data Locality** — scheduling principle for minimizing data movement and locating tasks near existing inputs.  
    https://distributed.dask.org/en/stable/locality.html
12. **wazero documentation** — pure-Go WebAssembly runtime; the host-import surface determines what a module can access.  
    https://wazero.io/docs/  
    https://github.com/tetratelabs/wazero
13. **Go Security Best Practices** — vulnerability checking and fuzzing practices.  
    https://go.dev/doc/security/best-practices
14. **SHEYTAN-LA repository source baseline** — primary code and product architecture used for the repository audit.  
    https://github.com/SHEYTANx13/SHEYTAN-LA/tree/9061e0768c99918e78f656c78cecddbe13e85892
15. **SHEYTAN-LA Architecture documentation** — one-authority runtime design and existing subsystem boundaries.  
    https://github.com/SHEYTANx13/SHEYTAN-LA/blob/9061e0768c99918e78f656c78cecddbe13e85892/docs/ARCHITECTURE.md
16. **SHEYTAN-LA Roadmap** — current delivered/future capability distinctions, including automatic Goal fan-out.  
    https://github.com/SHEYTANx13/SHEYTAN-LA/blob/9061e0768c99918e78f656c78cecddbe13e85892/docs/ROADMAP.md
17. **SHEYTAN-LA Agent Operating Manual** — repository-specific branch, verification, documentation and no-remote-mutation rules.  
    https://github.com/SHEYTANx13/SHEYTAN-LA/blob/9061e0768c99918e78f656c78cecddbe13e85892/docs/agent.md
18. **SHEYTAN-LA Security Policy** — current local IPC/API and local-first security boundary.  
    https://github.com/SHEYTANx13/SHEYTAN-LA/blob/9061e0768c99918e78f656c78cecddbe13e85892/SECURITY.md

---

**Parsa Tak (SHEYTAN)**  
SHEYTAN-LA — Mesh Computing Research and Implementation Blueprint  
22 November 2026
