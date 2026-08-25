# World Model Builder — Non-Functional Requirements (Pothole/Road-Surface Feature)

## Scope

This document covers timing, compute-complexity, and bandwidth requirements for this feature's contribution to World Model Builder — the Live/Prior layer fusion (world_model_builder_requirements.md, Sections A–F). It also resolves the two open items carried over from that document (WMB-REQ-03/WMB-REQ-10's staleness threshold, and WMB-REQ-06's transit-delay deadline).

These requirements trace to hara.md item #7 (World Model Builder real-time budget overrun, S2/E4/C2, **ASIL B**) — a different classification from the functional requirements' ASIL D, since item #7's consequence (delayed world-model output) is a lower-severity hazard class than items #1–#3's (corrupted/misleading fused output reaching Path Planner).

---

## A. Real-time execution budget

**WMB-NFR-01 [ASIL B]** — Fusion within World Model Builder's existing budget
This feature's Live+Prior fusion compute shall complete within World Model Builder's existing, externally-defined real-time (WCET) budget. This feature does not define that budget's absolute value — it is a pre-existing constraint World Model Builder already carries as a baseline item, and this feature's own contribution must be shown to fit within it, not generate a new budget of its own.

*Traces to:* hara.md item #7; fmea.md Finding 5.

**WMB-NFR-02 [ASIL B]** — Worst-case computational complexity bound
Fusing K simultaneous Live-layer candidates against a Prior-layer list of L cloud-advisory entries, each carrying an N×M depth-grid patch, shall have worst-case complexity that scales linearly with K, L, and N×M — i.e., proportional to the number of grid cells actually touched — not multiplicatively (K×L) as an object-matching or candidate-assignment approach would require.

*Rationale:* This is a direct, testable consequence of the grid-based design (WMB-REQ-09's "no explicit candidate correlation" property) — it must be verified as an actual property of the implementation, not merely assumed to follow from the architecture description. An implementation that reintroduces object-level matching internally (e.g., to deduplicate or cross-reference candidates before writing to the grid) would silently violate this even while superficially matching the interface contract, which is exactly the O(N×M) matching-complexity problem grid-based fusion was adopted to avoid.

*Traces to:* hara.md item #7; fmea.md Finding 5; world_model_builder_requirements.md WMB-REQ-09.

**WMB-NFR-03 [ASIL B]** — Inbound Prior-layer bandwidth
Receiving a cloud-advisory list of size L (interfaces.md #2) over its IPC/shared-memory channel shall not itself become a bottleneck independent of the compute bound in WMB-NFR-02 — i.e., transport time for a large L must be accounted for alongside fusion compute time when validating the overall budget in WMB-NFR-01, not treated as negligible by assumption.

*Traces to:* hara.md item #7.

---

## B. Update / cycle rate

**WMB-NFR-04 [ASIL B]** — Fusion cycle rate
Fusion shall execute at least once per Live-layer update cycle (matching Perception's own output rate, interfaces.md #1). The Prior layer is not required to have its own independent cycle rate — its content persists unchanged between its own event-driven updates (WMB-REQ-10) and participates in whatever fusion cycle the Live layer drives.

*Testable target:* documented camera-based AV perception systems typically operate at 10–30 Hz (33–100 ms cycle time) [12]. Fusion's own cycle rate shall not be the bottleneck below this range — verified by confirming fusion execution time (WMB-NFR-01/02) fits within the shortest cycle time Perception is actually configured to run at, not assumed from the range alone.

*Traces to:* world_model_builder_requirements.md WMB-REQ-10.

---

## C. Staleness and deadline thresholds — resolving the two open items

**WMB-NFR-05 [ASIL B]** — Prior-layer already-passed exclusion

A Prior-layer entry whose position lies behind the vehicle's current direction of travel — i.e., the vehicle has already passed it — shall be excluded from fusion, regardless of whether it otherwise falls within the ego-centric grid's spatial extent (WMB-REQ-03).

*Mechanism:* using the vehicle's current position and heading (both already available per Baseline Dependency E), compute the entry's position relative to the vehicle, then the dot product of that relative vector with the heading vector. A negative result means the entry lies behind the vehicle along its direction of travel; exclude it. This is a direction-aware refinement of WMB-REQ-03's positional-relevance check, not a duplicate of it — the existing check asks "is this within the grid window," which can include some rearward extent; this asks "is this still ahead of me," which is what actually determines whether the entry remains actionable for Path Planner.

*Why a fixed staleness cutoff is not used:* a fixed time-based cutoff was considered as an alternative mechanism to guard against fusing a Prior-layer entry describing a pothole that had since been repaired. That concern is already addressed elsewhere in this design, on an ongoing basis, without an arbitrary cutoff: the Healing Engine's fleet-level aggregation (block_diagram.md Section 5) continuously re-confirms or clears entries as vehicles pass and report fresh observations — a location nobody has driven past recently simply hasn't accumulated new evidence either way, and one that has been repaired will accumulate disconfirming observations from real traffic over time. A residual case not caught by aggregation (a repaired pothole on a rarely-traveled road, still marked active) reduces to the same bounded, comfort-only outcome this document has established throughout — an unnecessary avoidance maneuver for a hazard that's no longer there, not a new safety risk (hara.md's established reasoning). A fixed time-based cutoff would be solving a problem this design already handles elsewhere, at the cost of an arbitrary, hard-to-justify number; the direction-of-travel check solves a genuine, distinct problem (wasted fusion effort on unreachable content) with a mechanism that needs no arbitrary constant at all.

*SOTA basis:* Computing a heading-oriented region of interest — a window centered on and rotated to the ego-vehicle's direction of travel, rather than a simple radius — is documented practice for ego-centric map processing in autonomous driving pipelines [17]. This requirement applies the same underlying idea (heading-relative spatial relevance) as an exclusion filter rather than a windowing shape, but the geometric basis is the same.

*Traces to:* world_model_builder_requirements.md WMB-REQ-03, Open Items. This requirement addresses fmea.md Finding 4's underlying concern (acting on outdated cloud data) for the Prior layer via direction-of-travel exclusion plus the Healing Engine's ongoing aggregation, not a dedicated time-based cutoff. fmea.md and hara.md should be updated to reflect this, since Finding 4 as currently written asks for a staleness threshold, which is not the approach taken here.

**WMB-NFR-06 [ASIL B]** — Live-layer transit-delay deadline (resolves WMB-REQ-06's open item)

A Live-layer entry whose elapsed time since its timestamp exceeds **250 ms** shall be treated as a transit-delay failure and trigger WMB-REQ-06's second fallback trigger (exclude Live layer, use Prior layer alone), independent of `perception_health_state`.

*Basis:* Derived from documented AV perception-pipeline latency figures, combined per AUTOSAR's own configured-margin approach (nominal period + tolerance) rather than a single cited value for this exact deadline. Camera-based AV perception is documented as typically operating at 10–30 Hz (33–100 ms nominal cycle time) [12]; a real deployed cooperative-perception pipeline reports an optimized 80 ms end-to-end latency [15]; a separate real-world perception system measured 150–250 ms end-to-end latency under normal-to-degraded conditions and explicitly adopted 200 ms as its own operating upper bound [16]. Taking 100 ms as a conservative nominal cycle time and sizing the margin to the higher end of the documented worst-case range (150–250 ms) gives 250 ms as the deadline — tight enough to catch a genuinely delayed message, loose enough that a competently-optimized pipeline (80 ms, per [15]) clears it with margin to spare.

*Traces to:* world_model_builder_requirements.md WMB-REQ-06, Open Items.

---

## References

[12] "All You Need for Object Detection: From Pixels, Points, and Prompts to Next-Gen Fusion and Multimodal LLMs/VLMs in Autonomous Vehicles" — documents typical camera (30–60 Hz), LiDAR (10–20 Hz), and radar (10–30 Hz) update rates in AV perception. https://arxiv.org/pdf/2510.26641
[15] "AutoCast: Scalable Infrastructure-less Cooperative Perception for Distributed Collaborative Driving" — reports an optimized 80 ms end-to-end perception-and-planning pipeline latency, measured over 6000 frames. https://arxiv.org/pdf/2112.14947
[16] "A Real-Time Bike-Pedestrian Safety System with Wide-Angle Perception and Evaluation Testbed for Urban Intersections" — measures 150–250 ms end-to-end perception latency under normal-to-degraded conditions; adopts 200 ms as its own operating upper bound. https://arxiv.org/pdf/2604.17046
[17] "Cheap and Deterministic Inference for Deep State-Space Models of Interacting Dynamical Systems" — computes an ego-centric map region of interest centered at and oriented according to the vehicle's heading, rather than a simple radius. https://arxiv.org/pdf/2305.01773

---

## Open items surfaced while drafting

- WMB-NFR-01's actual budget value (World Model Builder's pre-existing WCET allocation) is not visible from these documents — this is the one item where a specific number genuinely cannot be supplied without access to the platform's own timing specification, which is out of this feature's scope by design (the user's own stated principle: adhere to an existing limit, don't invent one). The testable condition (total fusion+transport time ≤ that externally-supplied value) is fully specified; only the value itself is external.
- WMB-NFR-02 is a scaling property (linear vs. multiplicative) that needs to be verified against the actual implementation via profiling under worst-case K/L/N×M — testable as written, but requires the implementation to exist first.
- WMB-NFR-06's 250ms deadline is derived from real, cited data but is not drawn from a single source stating this exact value for this exact application — flagged honestly as a reasoned derivation, not an authoritative figure, and should be revisited once actual Perception pipeline latency is measured rather than estimated.
- WMB-NFR-05 addresses the Prior layer's freshness concern via a direction-of-travel exclusion check rather than a staleness threshold — fmea.md Finding 4 still frames this as needing a time-based threshold and should be updated to reflect that a threshold-based approach is not used here.