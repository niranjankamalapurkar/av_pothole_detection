# World Model Builder — Requirements (Pothole/Road-Surface Feature)

## Scope

World Model Builder's basic layered-fusion capability is assumed to already exist and is not defined here (hara.md Section 2). The requirements below define only this feature's own contribution: what the Live and Prior layers must contain, and — the majority of what follows — what World Model Builder must do when one or both of those specific inputs is unavailable, invalid, or delayed. Per hara.md's Classification Summary, this logic executes inside World Model Builder without architectural isolation and is developed and verified at **ASIL D**.

Each requirement is tagged with its ASIL and traced to the HARA item or FMEA Finding it addresses. Real-time budget and bandwidth requirements are out of scope here — see `world_model_builder_non_functional_requirements.md`.

---

## A. Georeferencing (Prior layer)

**WMB-REQ-01 [ASIL D]** — Prior layer georeferencing
World Model Builder shall convert each cloud-advisory entry's absolute `patch_center_lat`/`patch_center_lon` into a position within its ego-centric grid, using the vehicle's current absolute position (Baseline Dependency E).

*SOTA basis:* Converting a global lat/lon into a local Euclidean grid position is a well-documented technique — a geographic projection (e.g., transverse Mercator, or an equivalent local-tangent-plane/UTM conversion) centered on a reference point, after which coordinates are handled in ordinary Euclidean space rather than spherical coordinates. This is explicitly how production AV mapping systems relate a global-frame map to a vehicle-local submap, and it is the same category of transform ROS Nav2's `costmap_2d` package performs via its `static_layer`, which ingests a georeferenced map and places it into the robot's local grid frame via a coordinate transform. (Nav2's package is named `costmap_2d` as a matter of that project's own terminology — this feature's own design is described as a multi-layer/multi-channel world representation, not a costmap, per block_diagram.md's Architectural Basis and reference [11].)

*Traces to:* hara.md item #8 (Prior-layer placement drift).

---

## B. Nominal fusion

**WMB-REQ-02 [ASIL D]** — Valid Live + valid Prior fusion
When both the Live layer and the Prior layer contain valid, current data for a given grid cell, World Model Builder shall fuse them into one pothole depth value (negative elevation offset) and associated confidence score per cell using its existing fusion mechanism.

*SOTA basis:* Elementwise probabilistic (Bayesian or evidential) combination of independently-sourced grid layers into one fused layer is the standard mechanism in multi-layer world-representation architectures — each layer independently reports its own belief about a cell, and the combined representation merges them. This feature does not define that combination algorithm (out of scope, per block_diagram.md Section 3) — it defines that this combination is expected to occur under nominal conditions, with the degenerate-input behavior in Section C taking precedence whenever it applies. This feature's Live and Prior layers populate a road-surface elevation channel specifically — documented multi-layer architectures carry elevation and cost/friction as alternative, coexisting layer content types (block_diagram.md reference [11]), so this fusion is not expected to require converting elevation values into a cost representation.

*Traces to:* hara.md items #2, #3 (Live/Prior-layer-population and fusion correctness).

---

## C. Degenerate input handling — data availability

This is the largest and most safety-relevant group. All requirements below implement the same underlying principle already stated in block_diagram.md Section 3: **a missing or invalid input must never be treated as a confident negative observation.**

**WMB-REQ-03 [ASIL D]** — Prior layer invalid, absent, or delayed
If the Connectivity Manager provides no Prior-layer data for a cycle (no communication received), or the received data is invalid, or receipt is delayed beyond a defined staleness threshold, World Model Builder shall exclude the affected Prior layer entries (or the entire layer, if the packet structure/checksum itself is invalid) from fusion for that cycle and proceed using the Live layer alone. This shall not be treated as, or logged as, a negative ("no pothole expected") observation.

Two independent checks determine whether a Prior-layer entry is usable, and both must pass — one is not a substitute for the other:
- **Positional relevance** (answers "is this entry even placeable right now"): compute the entry's position relative to the vehicle's current position (Baseline Dependency E) and determine whether it falls within the ego-centric grid's spatial extent, plus a buffer accounting for the tolerance of the lat/lon-to-grid conversion itself (WMB-REQ-01). An entry falling outside this range is excluded as not currently relevant — this check is essentially free, since it reuses the same conversion WMB-REQ-01 already performs.
- **Temporal freshness** (answers "is this entry's content still trustworthy"): the entry's timestamp compared against a staleness threshold (open item — see fmea.md Finding 4). An entry can be positionally within range and still be temporally stale (e.g., a location the entry describes may have been repaired since the entry was last updated) — positional relevance does not establish content freshness.

*Traces to:* hara.md item #3; fmea.md Finding 4.

**WMB-REQ-04 [Assumption / Baseline Dependency, not a requirement this feature implements]** — Continuous ego-position availability
This feature assumes Localization continuously maintains an ego-position estimate (lat/lon) between discrete position fixes, rather than only at the moment a fix is received. This assumption directly enables WMB-REQ-01 (georeferencing at any cycle) and WMB-REQ-03 (evaluating positional relevance and staleness against a continuously-available current position, not a stale one).

*SOTA basis:* Continuous position estimation between discrete GNSS fixes via inertial dead reckoning (propagating an Extended Kalman Filter state using IMU measurements between GPS updates) is standard, extensively documented automotive/robotics practice — not a technique this feature needs to invent or own, since it is Localization's existing responsibility (Baseline Dependency E).

*Traces to:* hara.md Assumptions (Section 3); not independently classified, since this is Localization's item, not this feature's.

**WMB-REQ-05 [ASIL D]** — Localization invalid or unavailable
If Localization is unavailable or reports low confidence, World Model Builder shall exclude the Prior layer from fusion for that cycle, since there is no reliable way to place a cloud-sourced absolute-coordinate entry within the ego-centric grid without a trustworthy ego position (WMB-REQ-01). This does not affect the Live layer, which requires no localization-dependent conversion to be placed in the grid (hara.md item #8) — Live-layer fusion proceeds normally under this trigger.

*Rationale:* Distinct from WMB-REQ-03 even though the resulting action (exclude Prior layer) is the same — WMB-REQ-03 triggers on the Prior *data* being bad; this requirement triggers on the *positioning capability needed to place it* being bad. Keeping these separate preserves root-cause traceability if this occurs.

*Traces to:* hara.md item #8.

**WMB-REQ-06 [ASIL D]** — Live layer unavailable or stale
World Model Builder shall exclude the Live layer from fusion for a cycle and proceed using the Prior layer alone (if valid per WMB-REQ-03/WMB-REQ-05) under either of two independent triggers:
- Perception provides no Live-layer data for a cycle (empty array), or `perception_health_state` is DEGRADED or FAULTED; or
- Perception's Live-layer output is present and `perception_health_state` reports NOMINAL, but the elapsed time since the entry's timestamp exceeds an expected deadline — i.e., the message itself arrived late (transit or queueing delay), independent of what Perception itself reports about its own health.

Neither trigger shall be treated as, or logged as, a negative observation.

*SOTA basis:* The second trigger is a deadline-supervision pattern — checking that an expected update arrives within a bounded time window, independent of the producer's self-reported status — standard in automotive functional safety (AUTOSAR Watchdog Manager's Alive/Deadline Supervision; ISO 26262 identifies aliveness and deadline supervision as software safety mechanisms specifically for temporal protection). This exists because `perception_health_state` alone cannot detect a message that is late due to downstream transit or queueing delay rather than a problem Perception itself is aware of.

**Scope note:** this requirement's fallback (Prior-layer-only, or no output at all per WMB-REQ-07) is scoped to this feature's own pothole/road-surface channel. If Perception's staleness reflects a vehicle-wide loss of perception (affecting obstacle or pedestrian detection, not just this feature), that is a baseline, vehicle-level safety concern this feature does not define a response to and has no basis to assert one for — the same discipline this document applies to Path Planner's internals.

*Traces to:* hara.md item #1 (Perception failure mode 1); fmea.md Finding 1.

**WMB-REQ-07 [ASIL D]** — Both layers invalid
If both the Live layer (per WMB-REQ-06's triggers) and the Prior layer (per WMB-REQ-03/WMB-REQ-05's triggers) are invalid, absent, or delayed for a cycle, World Model Builder shall skip fusing pothole/road-surface content into the world model entirely for that cycle. This reduces the vehicle to baseline (pre-feature) risk for that cycle, consistent with hara.md's established reasoning — it shall not produce any fused output, positive or negative, for the affected cells.

*Traces to:* hara.md item #6 (reduces-to-baseline reasoning); fmea.md Finding 2 (false-positive-from-missing-value direction — this requirement is the primary defense against that failure mode, since no output at all is produced when neither input is trustworthy).

---

## D. Degenerate input handling — resource availability

**WMB-REQ-08 [ASIL D]** — Compute/thermal/clock-constrained operation
Under a fail-operational condition, reduced clock frequency, or thermal-derating condition affecting World Model Builder's own compute budget (distinct from WMB-REQ-03/WMB-REQ-05's data-availability triggers — this is a resource-availability trigger), World Model Builder shall rely solely on the Live layer and exclude the Prior layer from fusion, regardless of whether valid Prior-layer data is available. This sheds the lower-priority, comfort/proactive-avoidance-only Prior-layer processing first, consistent with hara.md item #7's stated shedding priority, before any core safety-relevant layer (obstacles, lane boundaries) is affected.

*Rationale:* This requirement is distinct from WMB-REQ-03 even though the resulting behavior (Live-only) is the same — WMB-REQ-03 triggers on the *data* being bad; this requirement triggers on WMB's own *resources* being constrained, independent of whether the Prior data itself is perfectly valid.

*Traces to:* hara.md item #7 (shedding priority); fmea.md Finding 5.

---

## E. Multi-candidate representation

**WMB-REQ-09 [ASIL D]** — Multiple simultaneous anomalies
World Model Builder's grid-cell-based fusion shall represent multiple simultaneous road-surface anomalies without requiring per-anomaly object identity, explicit candidate correlation, or multi-candidate assignment logic. Each anomaly is represented by the grid cells it spatially occupies; distinct anomalies at distinct locations are inherently distinguished by cell position, not by an assigned identifier.

*Rationale:* This is a design property that follows from adopting grid-based fusion (block_diagram.md Section 3) rather than an independently testable behavior in the usual sense — included here so it is stated explicitly as a requirement the fusion design must preserve, not merely an incidental side effect.

---

## F. Cloud data synchronization

**WMB-REQ-10 [ASIL D]** — Event-driven Prior-layer updates
The Prior layer shall be updated when the Connectivity Manager provides new or changed cloud-advisory data, not on a fixed continuous cycle matched to the Live layer's update rate. World Model Builder shall correctly fuse an unchanged (stale-but-not-expired) Prior layer against a continuously-updating Live layer between cloud updates.

*SOTA basis:* Asynchronous, multi-rate fusion — where a high-frequency source updates continuously while a low-frequency source provides periodic or event-triggered context — is a documented pattern for exactly this kind of mismatch (already cited in block_diagram.md's Architectural Basis, reference [4]). This is the inbound-data analogue of the Pothole Observation Reporter's own outbound threshold/debounce logic (`pothole_observation_reporter_functional_requirements.md`) — both exist because the cloud side of this feature is inherently sparse and event-driven, not continuous, while the Live layer is continuous.

*Traces to:* fmea.md Finding 4 (staleness threshold — the same open question of "how old is too old" applies directly to this requirement's "unchanged Prior layer" case).

---

## Open items surfaced while drafting

- The temporal staleness threshold referenced in WMB-REQ-03 and WMB-REQ-10 is still undefined (fmea.md Finding 4) — positional relevance (WMB-REQ-03) is now a concrete, derivable check, but the separate question of "how old is too old" for content freshness remains open.
- The expected deadline referenced in WMB-REQ-06's second trigger (Live-layer transit delay) is not yet defined — needs a value derived from Perception's own expected update rate plus tolerance, analogous to AUTOSAR's Deadline Supervision configuration.
- WMB-REQ-04 depends on Localization's own dead-reckoning implementation actually existing and being adequate between fixes — this feature cannot verify that assumption itself.
- WMB-REQ-05 (Localization invalid) depends on Localization actually reporting a confidence/validity signal this feature can check — if no such signal exists on the Localization interface, this requirement cannot be implemented as written and the interface itself would need to be revisited.
- **Resolved** — "costmap" terminology corrected throughout: World Model Builder's output is a multi-layer/multi-channel world representation, not a single flat costmap. A documented patent describing exactly this kind of multi-layer grid names elevation and friction/cost as *alternative* content types for a ground-surface layer, not values requiring conversion into one another (block_diagram.md reference [11]). WMB-REQ-02's fused output stays depth-typed (per-cell elevation values plus a scalar confidence) — this feature's layers populate an elevation channel; if some other consumer of the world representation needs a cost/friction-typed channel, that is a separate, additional projection downstream of this feature's fusion, not a replacement for it. No open question remains here.