# Pothole Observation Reporter — Functional Requirements

## Scope

The Pothole Observation Reporter is the one component in this feature classified independently at **QM** (hara.md Classification Summary) — it reads World Model Builder's fused output but has no path back into any safety-critical chain, reporting only to the cloud. All requirements below are QM. Report-frequency and timing-budget requirements are out of scope here — see pothole_observation_reporter_non_functional_requirements.md.

---

## A. Input consumption

**POR-REQ-01 [QM]** — Elevation-channel consumption
The Reporter shall consume World Model Builder's output specifically via the road-surface elevation channel (interfaces.md #3) — the fused depth-grid patch and its confidence score — not the full multi-layer/multi-channel world representation broadly. It has no defined interface to, and no need to inspect, other channels (obstacle occupancy, lane boundary, etc.).

*Traces to:* block_diagram.md Architectural Basis point 1; interfaces.md #3.

---

## B. Candidate identification

**POR-REQ-02 [QM]** — Depression-depth threshold
A grid cell shall be treated as a pothole candidate only if its fused depth value represents a depression (negative elevation relative to the surrounding road-surface baseline) of magnitude greater than **25 mm**. Positive elevation values (raised surfaces — debris, speed bumps, curbing) are explicitly out of scope for this feature and shall not be reported.

*Open item this requirement surfaces:* this presupposes a sign convention (negative = below-baseline depression, positive = above-baseline) for `depth_values` that is not currently stated anywhere in interfaces.md. This must be established as part of interface #1/#2/#3's specification, not assumed here — flagged for correction in those documents.

*Basis:* FHWA's pavement distress severity classification defines potholes less than 25 mm deep as low severity, 25–50 mm as moderate, and greater than 50 mm as high severity [18]. 25 mm is used here as the minimum-candidacy threshold (matching FHWA's low-severity floor) rather than a higher severity band, since even low-severity potholes are within this feature's stated purpose (proactive comfort/avoidance, problem_statement.md) — the severity bands themselves may be useful for a future confidence/priority refinement but are not required for basic candidacy.

*Testable as:* inject a fused depth value of -24 mm (shall not be reported) and -26 mm (shall be reported) and verify the boundary is enforced exactly.

**POR-REQ-03 [QM]** — Minimum spatial extent
A candidate shall only be reported if the contiguous depressed area meets or exceeds FHWA's minimum plan dimension for a pothole — a 150 mm minimum diameter, or equivalently approximately 0.02 m² minimum area [19]. A single isolated depressed cell smaller than this, even if it exceeds the depth threshold (POR-REQ-02), is treated as sensor/surface noise, not a reportable candidate.

*Basis:* FHWA's Distress Identification Manual for the Long-Term Pavement Performance Program states a circular pothole must have a minimum diameter of 150 mm (a 150 mm circle must fit inside irregular-shaped ones) to be counted as a pothole in official distress surveys at all [19] — below this, it is not classified as a pothole regardless of depth.

*Testable as:* inject a depressed region of 140 mm diameter at -30 mm depth (shall not be reported, fails size threshold despite passing depth threshold) and 160 mm diameter at -30 mm depth (shall be reported).

*Traces to:* WMB-REQ-01's grid_resolution_cm (the grid resolution must be fine enough to resolve a 150 mm feature — a cross-reference worth confirming against whatever value is ultimately chosen for grid_resolution_cm).

---

## C. Communication behavior

**POR-REQ-05 [QM]** — Asynchronous, event-driven reporting
The Reporter shall not transmit on a periodic/heartbeat schedule. It shall transmit only when a new pothole candidate is identified (POR-REQ-02, POR-REQ-03) at a position not currently suppressed — see pothole_observation_reporter_non_functional_requirements.md POR-NFR-01 for the suppression-window timing budget behind this behavior.

*Traces to:* interfaces.md #4.

**POR-REQ-06 [QM]** — No independent geocoding
The Reporter shall use the `patch_center_lat`/`patch_center_lon` already present on World Model Builder's fused output (interfaces.md #3) directly in its report (interfaces.md #4). It shall not perform its own coordinate conversion, geocoding, or independent Localization lookup — that work is already done upstream, per Baseline Dependency E, and duplicating it here would be redundant.

*Traces to:* interfaces.md #3, #4, Baseline Dependency E.

---

## D. Degraded-source handling

**POR-REQ-07 [QM]** — Suppress reporting of Prior-only-derived candidates
The Reporter shall not report a candidate to the cloud if its fused value was derived solely from the Prior layer — i.e., World Model Builder's Live-layer-unavailable fallback (world_model_builder_requirements.md WMB-REQ-08) was active for that observation.

*Rationale:* such a report would be circular — the cloud already originated the data it would be reporting back. Worse, it risks being miscounted by the Healing Engine's aggregation logic as an independent confirming observation: interfaces.md #4's `vehicle_id` field distinguishes which vehicle sent a report, but not whether the report carries any genuinely new information. A Prior-only echo would look identical to a real independent observation to that aggregation logic, undermining the very independence assumption it depends on (fmea.md Finding 7).

*Depends on (not yet defined):* a signal on World Model Builder's fused output (interfaces.md #3) indicating whether the Live layer contributed to a given observation. This is narrower than the entry_type state machine removed earlier in this design's history — a single provenance bit, not a status classification, and it does not reintroduce any of the reasoning entry_type's removal was meant to close off.

**POR-REQ-08 [QM]** — Suppress reporting on World Model Builder fusion-health fault
If World Model Builder's fusion process itself is unhealthy — a fault distinct from a low `fused_confidence_score`, which represents a valid-but-uncertain result — the Reporter shall suppress all cloud reporting, regardless of what `fused_depth_values` or `fused_confidence_score` might otherwise contain.

*Rationale:* interfaces.md #3 currently exposes no health/status signal for World Model Builder's own fusion process, unlike Perception (`perception_health_state`, interfaces.md #1). Without an equivalent signal here, the Reporter has no way to distinguish a genuinely low-confidence result from stale or corrupted output following a fusion-process fault. Reporting the latter pollutes the fleet database with invalid data — worse than reporting nothing, consistent with this design's established principle (world_model_builder_requirements.md Section C) that degraded or missing input must never be treated as valid.

*Depends on (not yet defined):* a `fusion_health_state`-equivalent field on interfaces.md #3's payload, analogous to `perception_health_state`. Not currently present.

---



[18] FHWA pavement severity classification — potholes <25 mm depth: low severity; 25–50 mm: moderate; >50 mm: high severity (FHWA, 2003), as cited in "Autonomous pavement condition assessment" patent. https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/9196048
[19] Distress Identification Manual for the Long-Term Pavement Performance Program (Fifth Revised Edition), FHWA-HRT-13-092 — minimum 150 mm diameter / ~0.02 m² minimum area for a pothole to be counted in distress surveys. https://www.fhwa.dot.gov/publications/research/infrastructure/pavements/ltpp/13092/001.cfm

---

## Open items surfaced while drafting

- POR-REQ-02 depends on a sign convention for `depth_values` (negative = depression) that is not currently stated in interfaces.md — needs to be added there, not just assumed here.
- POR-REQ-03 depends on `grid_resolution_cm` (world_model_builder_requirements.md WMB-REQ-01) being fine enough to resolve a 150 mm feature — worth confirming once that value is set, since too coarse a resolution would make this requirement unenforceable.
- POR-REQ-07 depends on a Live-layer-contribution signal on interfaces.md #3 that does not currently exist.
- POR-REQ-08 depends on a fusion-health signal on interfaces.md #3 that does not currently exist.
- POR-REQ-07 and POR-REQ-08 together suggest interface #3's payload needs two new fields, not one — worth adding both in the same pass through interfaces.md and world_model_builder_requirements.md, rather than addressing them separately, since both originate from the same underlying gap: World Model Builder's own output currently carries no provenance or health information about itself, only about the result.