## Scope

This is an independent, bottom-up Failure Mode and Effects Analysis (FMEA) of the Pothole Detection feature, built against the current architecture (block_diagram.md, interfaces.md) — layer-based fusion into the World Model Builder, no dedicated matching/verification components, no physical-confirmation sensor.

---

## Method

Failure mode: what fails, in one line.
Category: Loss of function / Erroneous output / Delayed output / Intermittent operation.
Local effect: what happens to that block's own output.
System effect: what happens downstream.
Coverage: COVERED — [document, section] if addressed, or NOT COVERED — see Finding N.

---

## Perception (includes pothole detection)

**1. Failure mode: Complete loss of function**
    Category:      Loss of function
    Local effect:  Empty candidate array; perception_health_state reflects the degradation.
    System effect: Correct behavior depends entirely on World Model Builder's degraded-input handling correctly reading perception_health_state and falling back to the Prior layer alone, rather than treating the empty array as a confident negative observation.
    Coverage:      Requirement stated (block_diagram.md Section 3) — implementation correctness NOT COVERED, see Finding 1

**2. Failure mode: Visual misclassification (ghost pothole)**
    Category:      Erroneous output
    Coverage:      COVERED — hara.md item #4

**3. Failure mode: Miss on a real pothole**
    Category:      Erroneous output (false negative)
    Coverage:      COVERED — hara.md item #6

**4. Failure mode: Compute-resource contention with primary object detection**
    Category:      Delayed output / partial loss of function
    Local effect:  Pothole classification competes with primary object detection for shared compute.
    System effect: Perception's own internal prioritization drops pothole classification first, per block_diagram.md Section 1.
    Coverage:      COVERED — block_diagram.md Section 1

**5. Failure mode: Pothole content may not warrant a separately-defined Perception output at all**
    Category:      Design/architecture question, not a component failure
    Local effect:  interfaces.md #1 defines a dedicated pothole-candidate feed from Perception to the Live layer, distinct from Perception's general object/lane output.
    System effect: Documented occupancy-based perception (e.g., BEVFusion, already cited in block_diagram.md) produces one unified semantic/occupancy output covering all classes together, rather than separate per-class channels — described in the reviewed literature as the more current pattern, with per-subtask separate pipelines noted as a limitation being moved away from. Whether road-surface/pothole content should instead be one semantic class within Perception's already-unified, already-baseline output — removing interface #1 as a separately-defined feed entirely — is an open question this feature has not resolved.
    Coverage:      NOT COVERED — see Finding 6

---

## Localization

**1. Failure mode: Drift or complete loss of position fix**
    Category:      Erroneous output (drift) / Loss of function (total loss)
    Local effect:  World Model Builder's placement of the Prior layer's grid patch within its ego-centric representation becomes unreliable — converting the cloud's absolute-coordinate entry into a position in the vehicle's current grid requires an accurate ego position.
    System effect: This does not affect the Live layer's placement in the same way — Perception's output arrives already ego-relative (a distance measurement), needing no localization-dependent conversion to sit correctly in the grid. Localization only affects Live-layer-derived content later, when the Reporter exports an absolute position for the cloud report (see Pothole Observation Reporter, Finding 2 below).
    Coverage:      COVERED — hara.md item #8, interfaces.md Baseline Dependency E

---

## World Model Builder — Live layer population, Prior layer population, fusion, degraded-input handling

**1. Failure mode: Live-layer-population logic transcribes Perception's output incorrectly**
    Category:      Erroneous output (malfunction, HARA category)
    Local effect:  A defect in this feature's own logic causes the fused grid to misrepresent what Perception actually detected, independent of whether Perception's detection itself was correct.
    System effect: Executes inside World Model Builder's own processing without isolation (hara.md Section 2) — must be developed and verified at ASIL D directly.
    Coverage:      NOT COVERED — see hara.md item #2 and Finding 2

**2. Failure mode: Degraded-input-handling logic misreads absence in either direction**
    Category:      Erroneous output (malfunction, HARA category)
    Local effect:  A defect causes the fused output to either (a) treat a missing/degraded layer as a confident negative observation, or (b) misread a missing value as a valid positive reading.
    System effect: Direction (a) reduces to baseline risk if the other layer also has no information (hara.md item #6's reasoning). Direction (b) is the more consequential case — a false-positive fused output reaching Path Planner via the world model. Both directions require this logic to be verified at ASIL D (hara.md Section 2), not treated as a bounded QM concern.
    Coverage:      NOT COVERED — see hara.md item #3 and Finding 3

**3. Failure mode: Cloud-advisory list staleness has no defined threshold**
    Category:      Erroneous output
    Local effect:  interfaces.md #2 carries a timestamp on the Prior-layer feed, but no staleness threshold is defined for when the degraded-input-handling logic should treat it as unavailable.
    System effect: World Model Builder could fuse a Prior-layer patch that no longer reflects current road conditions as if it were current.
    Coverage:      NOT COVERED — see Finding 4

**4. Failure mode: Own real-time budget for grid-patch fusion is undefined, with no shedding priority under load**
    Category:      Delayed output
    Local effect:  Elementwise fusion of two N×M depth-grid patches, potentially for multiple simultaneous candidates and a large cloud-advisory list, has a compute cost that scales with patch size and candidate count.
    System effect: No worst-case complexity bound is defined for this feature's contribution to World Model Builder's existing real-time budget, and no shedding priority is defined for sustained budget pressure (including fail-operational or reduced-clock conditions) — unlike Perception, which already states pothole classification is dropped first under its own compute pressure (block_diagram.md Section 1), World Model Builder has no equivalent statement for this feature's two layers.
    Coverage:      NOT COVERED — see hara.md item #7 and Finding 5

---

## Pothole Observation Reporter

**1. Failure mode: Threshold/debounce logic fails to trigger a report for a genuine, significant confidence change**
    Category:      Erroneous output (false negative) / Loss of function
    Local effect:  A real, fused hazard observation never gets reported to the cloud.
    System effect: Bounded by the same reasoning as hara.md item #6 — a single vehicle's missed report reduces the system to baseline risk for that observation. No path back into the safety-relevant chain regardless.
    Coverage:      COVERED — hara.md Classification Summary (Reporter's QM disposition)

**2. Failure mode: Reported position is stale relative to when the underlying Live-layer observation was made**
    Category:      Erroneous output (minor)
    Local effect:  If the vehicle has moved meaningfully between the original Live-layer detection and the moment the Reporter exports an absolute position (using World Model Builder's Localization access, Baseline Dependency E), the reported coordinate could lag the true detection location.
    System effect: A minor accuracy concern for the cloud database — the Reporter's output never reaches Path Planner or the world model (hara.md Classification Summary), so this is bounded, not a real-time hazard.
    Coverage:      NOT COVERED as a dedicated finding — low severity given the confirmed dead-end output path; noted for completeness, no action item.

**3. Failure mode: Threshold/debounce logic over-triggers, reporting excessively**
    Category:      Erroneous output / Intermittent operation
    Local effect:  Frequent, low-value reports sent to the cloud.
    System effect: A bandwidth/telemetry-burst concern (hara.md item #5), not a real-time vehicle-safety concern.
    Coverage:      COVERED — hara.md item #5

---

## Connectivity Manager / Local Datalogger / Cloud (Map Update & Healing Engine)

**1. Failure mode: Network-down fallback fails; intermittent/partial/corrupted data**
    Category:      Loss of function / Intermittent operation
    System effect: Bounded — the Reporter's output has no path back into the world model or Path Planner, so cloud-side failures cannot elevate real-time risk. Intermittent/partial data is rejected outright per interfaces.md #2.
    Coverage:      COVERED — hara.md Classification Summary, interfaces.md #2

**2. Failure mode: Healing Engine's multi-report aggregation logic is unspecified, for both adding and clearing entries**
    Category:      Design gap (external system, not a vehicle item)
    Local effect:  block_diagram.md Section 5 states aggregation is "weighted by independence," but the threshold, time window, and what counts as an independent report are undefined.
    System effect: With no onboard physical-confirmation sensor anywhere in this design, fleet-level aggregation is the only mechanism distinguishing a genuine, persistent hazard from a correlated false positive or a correlated false negative. Diversity in reporting time and reporting vehicle are the only signals currently available; whether additional context (weather, lighting) is needed is unmodeled.
    Coverage:      NOT COVERED — see Finding 7

---

## World Model Builder → Path Planner / Vehicle Controls (baseline — out of scope)

**Note:** this feature has no direct interface with Path Planner or Vehicle Controls, and this document does not assert anything about their internal validation mechanisms.

---

## Findings Summary

**Finding 1 —** Degraded-input-handling logic correctness is unverified (absence-as-negative direction)

block_diagram.md Section 3 states the required behavior, but nothing verifies the actual implementation does this correctly.

Action: verify this logic against the actual design and implementation; ASIL D priority (hara.md Section 2).

**Finding 2 —** Live-layer-population logic correctness is unverified

Whether the transcription of Perception's output into the Live layer's grid representation is faithful to what Perception actually detected is unverified.

Action: verify this logic against the actual design and implementation; ASIL D priority (hara.md Section 2).

**Finding 3 —** Missing-value-as-real-value misinterpretation (false-positive direction) is not ruled out

Whether a null/absent Live or Prior layer entry could be misread by fusion as a real, low-depth observation is not ruled out by the current design.

Action: specify explicit, unambiguous "no data" handling in the fusion/degraded-input logic — distinct from any numeric value, including zero.

**Finding 4 —** Cloud-advisory list staleness threshold undefined

interfaces.md #2 carries a timestamp, but nothing states how old is too old for the degraded-input-handling logic to treat it as unavailable.

Action: define a staleness threshold, past which the Prior layer is treated as absent rather than authoritative.

**Finding 5 —** World Model Builder's grid-patch fusion has no bounded compute cost and no shedding priority under load

No worst-case complexity bound is defined for N×M patch fusion, and no priority order is stated for sustained real-time budget pressure — Perception already states its own priority (pothole classification dropped first); World Model Builder's fusion budget has no equivalent statement.

Action: define a worst-case complexity bound, confirm it fits within World Model Builder's existing WCET allocation, and state explicitly that the Prior (cloud) layer is dropped first under sustained pressure, before any core safety layer is affected.

**Finding 6 —** Whether Perception should output a dedicated pothole-candidate feed at all is unresolved

Documented occupancy-based perception (e.g., BEVFusion, already cited) produces one unified semantic/occupancy output rather than separate per-class channels — the current dedicated-feed design (interfaces.md #1) may not reflect this.

Action: determine whether interface #1 should be removed entirely, with road-surface content instead treated as one semantic class within Perception's already-unified, already-baseline output.

**Finding 7 —** Healing Engine's aggregation-independence criteria are unspecified

The cloud's definition of "independent" vs. "correlated" reports — for both adding and clearing entries — is undefined. With no onboard physical-confirmation mechanism in this design, this is the sole defense against both correlated false positives and correlated false negatives at fleet scale.

Action: define the aggregation logic explicitly, including a concrete, testable definition of independence (time-of-day diversity, vehicle/sensor diversity, and whether weather/lighting context is needed).