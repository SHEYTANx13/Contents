**SHEYTAN-LA Research & Development Report**  
**Title:** Architecture Decision Record — Fastest Local Agent Platform (2026)  
**Version:** 2026-10-11  
**Status:** Decision-ready  
**Author:** Parsaetak R&D Agent

---

### 1. Executive Summary

SHEYTAN-LA is evolving from a capable local-first laboratory into the **fastest practical local agent application**.  

The core decision is clear:

- **Go remains the main orchestrator** — the single authority for run lifecycle, tools, memory, sessions, Governor, and evidence.
- **C++ owns the performance-critical compute path** (inference engine).
- **C# is used selectively** only where it delivers measurable speed or deeper Windows integration.
- Two viable UI paths exist. After evaluation, the recommended primary path is **Go + C++ + Qt** for long-term strength across operating systems, with a **Windows-specific WinUI 3 shell** available as a high-performance alternative if pure Windows dominance is prioritized.

This report defines the target architecture, language boundaries, UI choice, migration sequence, and measurement framework so the product can systematically become the fastest local agent without sacrificing the existing design integrity (one authority per concern, evidence over claims, local-first, honest degradation).

---

### 2. Objectives and Success Criteria

**Primary Objective**  
Deliver the lowest end-to-end latency and highest sustained responsiveness for a local AI agent on consumer hardware, with Windows as the first-class platform.

**Success Criteria**
- Cold start to interactive UI + engine ready: target under 0.8 s on mid-range Windows hardware.
- Time-to-first-token (warm engine): competitive with or better than current managed llama.cpp path.
- Streaming update latency: under 16 ms from token arrival to visible update.
- Idle memory footprint: significantly lower than current WebView-based shell.
- Tool and file output paths remain low-overhead and sandboxed.
- Architecture stays maintainable: Go continues to own all high-level policy and lifecycle.
- Multi-OS viability is preserved (Linux and macOS remain first-class citizens under the Qt path).

---

### 3. Current State Assessment

The existing system already contains strong foundations:

- Sophisticated Go runtime with one authority per concern (engine lifecycle, run registry, tool registry, memory, Governor, sessions).
- Managed llama.cpp path + a real native C++ engine (`native/engine`) with forward pass, KV cache, tokenizer, and parity tests.
- Evidence-based GPU/accelerator claims and cooperative resource protection.
- Extensive deterministic tests, race gates, and honest documentation culture.

Remaining speed limiters are primarily:

- WebView-based UI shell (Wails) introduces bridge and rendering overhead.
- Native C++ engine is still treated as an alternative rather than the primary path.
- Some orchestration and streaming paths still cross process or language boundaries more than necessary.

The next phase must remove these limiters while protecting the existing strengths.

---

### 4. Performance Goals (Windows Reference)

| Metric                        | Current Approximate | Target                  | Priority |
|-------------------------------|---------------------|-------------------------|----------|
| Cold start to interactive     | 1–2 s               | < 0.8 s                 | P0      |
| TTFT (warm, small prompt)     | Variable            | Lowest practical        | P0      |
| Streaming visual latency      | WebView limited     | < 16 ms                 | P0      |
| Idle process memory           | 80–150 MB+          | < 80 MB preferred       | P1      |
| Tool / file write overhead    | Acceptable          | Minimal measurable      | P1      |
| Binary / installer size       | Medium              | Competitive             | P2      |

All targets will be measured on real Windows hardware with representative models and workloads.

---

### 5. Technology Evaluation Summary

**Orchestration Layer**  
Go remains the strongest practical choice for the main agent loop, concurrency, streaming, tool routing, and long-running reliability. It already owns the sophisticated authorities that define SHEYTAN. Replacing it would destroy more value than it creates.

**Compute / Inference Layer**  
C++ is the correct owner of the heavy numerical path. The existing native engine should be promoted to primary. Managed llama.cpp stays as the reliable fallback. Selective use of C# is acceptable only for thin Windows-specific helpers where the measurement justifies the extra language.

**UI Layer — Two Contenders**

| Criterion                  | Go + C++ + Qt                          | WinUI 3 (C#)                          |
|----------------------------|----------------------------------------|---------------------------------------|
| Raw UI responsiveness      | Excellent (native rendering)           | Excellent (DirectX / Composition)     |
| Cross-platform             | Strong (Windows, Linux, macOS)         | Windows only                          |
| Integration with Go core   | Clean via IPC or thin bindings         | Requires IPC or mixed-language host   |
| Long-term maintenance      | One UI technology across OS            | Platform-specific shell               |
| Maximum Windows polish     | Very high                              | Highest possible                      |
| Team / complexity cost     | C++ knowledge required                 | C# + WinUI knowledge required         |
| Alignment with “own engine”| Excellent                              | Good                                  |

**Recommendation**  
Adopt **Go + C++ + Qt** as the primary architecture.  

It delivers near-native speed on Windows while keeping a single, strong codebase for Linux and macOS. A dedicated WinUI 3 shell remains a valid future option if pure Windows market dominance becomes the overriding goal, but it should not be the default path.

---

### 6. Recommended Target Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                        │
│              Qt (C++ / QML) — primary                        │
│         Optional: WinUI 3 shell for pure Windows             │
└──────────────────────────┬──────────────────────────────────┘
                           │ High-performance IPC
                           │ (shared memory / named pipes /
                           │  zero-copy where possible)
┌──────────────────────────▼──────────────────────────────────┐
│                 Go Orchestration Core                        │
│  Run lifecycle • Tool registry • Memory • Governor           │
│  Sessions • Evidence ladder • Config • Agent loop            │
│  (Single process, single authority model preserved)          │
└──────────────────────────┬──────────────────────────────────┘
                           │ Typed, coarse-grained boundary
┌──────────────────────────▼──────────────────────────────────┐
│              C++ Inference Engine (Primary)                  │
│  Forward pass • KV • Tokenizer • Sampler • Hardware          │
│  Existing native engine promoted and further optimized       │
│  Managed llama.cpp retained as fallback                      │
└─────────────────────────────────────────────────────────────┘
```

**Language Ownership Rules**
- **Go**: All policy, lifecycle, orchestration, tool execution logic, evidence, and state.
- **C++**: All numerical inference work and any future ultra-low-latency helpers that measurements prove necessary.
- **C#**: Used only when a Windows-specific component is measurably faster or required for deep system integration. Never becomes the main orchestrator.

---

### 7. UI Path Decision

**Primary Path: Qt**  
- One technology for all operating systems.  
- Native performance characteristics on Windows, Linux, and macOS.  
- Direct integration path with the existing C++ engine.  
- Avoids the WebView ceiling that currently limits streaming and UI responsiveness.  
- Keeps the product coherent as a true multi-OS laboratory.

**Secondary / Optional Path: WinUI 3**  
- Available if future product strategy prioritizes absolute maximum Windows experience and Store/integration depth over cross-platform uniformity.  
- Would sit as an alternative shell talking to the same Go + C++ core via the same IPC contract.

Decision: Proceed with Qt as the default UI technology for the next major cycle.

---

### 8. Migration Roadmap

**Phase 0 — Measurement Baseline (immediate)**  
Instrument current cold start, TTFT, streaming latency, and memory on real Windows hardware. Establish numbers that every subsequent change must beat.

**Phase 1 — Promote Native Engine (high leverage)**  
- Make the existing C++ engine the preferred backend when it can generate.  
- Complete remaining arithmetic optimizations under the existing parity gates.  
- Tighten the evidence ladder and TTFT path.  
- Keep managed llama.cpp as automatic fallback.

**Phase 2 — Native UI Shell**  
- Introduce Qt presentation layer.  
- Connect it to the unchanged Go orchestration core via a clean, high-performance IPC boundary.  
- Preserve all existing run, tool, memory, and Governor behavior.  
- Remove the WebView dependency for the primary product.

**Phase 3 — Refinement & Optional Windows Shell**  
- Further reduce memory and latency based on real measurements.  
- Evaluate a thin WinUI 3 shell only if Windows-specific gains justify the extra surface.  
- Continue expanding the C++ engine capabilities under the same ownership model.

**Phase 4 — Multi-OS Polish**  
- Ensure Linux and macOS paths remain first-class under the Qt shell.  
- Apply the same performance discipline to every platform.

Each phase ends only when measured results meet or exceed the targets defined in Section 4.

---

### 9. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Qt learning and packaging cost | Medium | Start with a thin shell; keep Go core untouched |
| IPC latency between UI and Go | Medium | Prefer shared memory / zero-copy designs; measure early |
| C++ engine optimization introduces numerical drift | High | Mandatory parity and bit-for-bit gates before any merge |
| Scope expansion during migration | High | Strict phase gates; no new features until performance targets are met |
| Team language split | Medium | Clear ownership rules documented and enforced |

---

### 10. Measurement Framework

Every change must be evaluated against:

- Cold-start time (UI interactive + engine ready)
- TTFT under fixed prompts and models
- Streaming visual latency
- Idle and peak memory
- Tool round-trip latency
- File output throughput

Results are recorded in the worklog and become part of the release evidence. No performance claim is accepted without numbers from real hardware.

---

### 11. Conclusion

The fastest path forward is evolutionary, not revolutionary:

- Keep Go as the intelligent orchestrator.  
- Elevate the existing C++ engine to primary status and continue optimizing it.  
- Move the presentation layer to Qt for native speed and true multi-OS strength.  
- Use C# only when measurements prove a clear, localized advantage.  

This combination preserves the architectural integrity that already makes SHEYTAN distinctive while systematically removing the remaining speed limiters. The result will be a stronger, faster, more coherent local agent platform that remains Windows-first without becoming Windows-only.

**Immediate next actions**
1. Establish quantitative baselines on real Windows hardware.  
2. Promote the native C++ engine to preferred path under the existing test gates.  
3. Prototype the Qt shell against the current Go process using a minimal IPC contract.  
4. Update this document with measured results after each phase.

This report is the new authoritative Research & Development baseline for SHEYTAN-LA.
