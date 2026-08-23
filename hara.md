## Scope
This document is limited to hazard identification and ASIL classification for **functional safety (ISO 26262)**. Items that trace to **SOTIF (ISO 21448)** are identified and separated out, but not classified with an ASIL — they require scenario-based residual-risk acceptance, a different process, out of scope here. Items that are pure vehicle/business impact (no path to human harm) are excluded from HARA entirely. Safety mechanisms and architectural allocation belong in the downstream Functional Safety Concept (FSC).

---

## 1. Purpose

This feature integrates road-surface/pothole content directly into the World Model Builder's existing layered-costmap fusion, and reports observations to the cloud for fleet-level aggregation. This document determines:

1. Which existing item(s) this feature modifies, and whether it introduces a separately classifiable item of its own.
2. Whether the feature introduces functional-safety hazardous events not already covered by existing analysis (vs. SOTIF, vs. no safety relevance).
3. ASIL classification for any hazardous events, derived from S/E/C — not asserted.

This is a Delta HARA: it assumes the baseline vehicle-level HARA (dynamic driving task, lane keeping, collision avoidance) already exists and is unchanged.

---

## 2. Item Definition (Delta)

**World Model Builder**
- Status: Existing item — MODIFIED. This feature's Live-layer population logic and degraded-input handling behavior (block_diagram.md Section 3) execute as an integral, non-isolated part of World Model Builder's own fusion process. There is no separate component and no validation step specific to this feature's contribution standing between this logic and World Model Builder's core processing.
- ASIL: Assumed D, inherited from World Model Builder's existing role feeding Path Planner's safety-critical decisions — not independently re-derived here. Per ISO 26262 coexistence principles, software integrated into a higher-ASIL element without demonstrated freedom from interference must be developed to that element's ASIL. This feature's world-model-facing logic is therefore developed at ASIL D — it is not a separately classifiable lower-ASIL item.

**Perception**
- Status: Existing item. Pothole-candidate generation is one more object class within Perception's existing pipeline — not a new item, and never a candidate for separate QM classification; it shares whatever development rigor Perception's existing safety-relevant object detection already carries. Whether this content should even be a *separately defined* output from Perception, as opposed to one semantic class within Perception's already-unified occupancy/semantic output, is an open architectural question — see Finding 6.
- ASIL: Unchanged, inherited from Perception's existing classification (not independently re-derived here).

**Localization**
- Status: Existing item, unaffected in isolation. Feeds World Model Builder generally (Baseline Dependency E), not specifically for this feature.
- ASIL: Unchanged.

**Path Planning & Trajectory Arbitration**
- Status: Existing item — UNAFFECTED DIRECTLY. No interface with this feature at all; consumes only the World Model Builder's already-fused output. This document does not assert anything about Path Planner's own internal validation mechanisms, and does not rely on any such mechanism for this feature's classification.
- ASIL: D (unchanged)

**Vehicle Controls**
- Status: Existing item — UNAFFECTED.
- ASIL: D (unchanged)

**Connectivity Manager (baseline)**
- Status: Existing item. Assumed to authenticate and validate all incoming cloud data before forwarding to any consumer (Baseline Dependency D).
- ASIL: Not independently derived here.

**Pothole Observation Reporter (new)**
- Status: New item, architecturally separate from World Model Builder — reads its fused output but has no path back into any safety-critical chain. Reports only to the cloud, via the Connectivity Manager.
- ASIL: Under determination — Section 6.

**Cloud Infrastructure (Map Update & Healing Engine, Central Pothole Database) — external, not a vehicle item**
- Status: Out of this HARA's scope by definition. Its output reaches the vehicle only as externally-sourced data via the Connectivity Manager.
- ASIL: N/A (external system).

---

## 3. Assumptions Carried from Baseline HARA

- Baseline vehicle-level hazards (unintended lateral trajectory violation, unintended braking/acceleration) already have an existing Safety Goal, referred to here as SG-01.
- The baseline S/E/C ratings for those hazards do not change due to this feature — inherited from the baseline item, not re-derived here.
- World Model Builder is assumed ASIL D — not independently re-derived here, and now the primary place this feature's own development rigor is anchored (Section 2).
- The Connectivity Manager is assumed to authenticate and validate all incoming cloud data before forwarding to any consumer — inherited baseline architecture.
- This document does not assume the existence of any specific downstream trajectory-validation mechanism (in Path Planner or elsewhere) as a defense for this feature's hazards. If such a mechanism exists in the broader vehicle architecture, it would provide additional defense in depth, but this document's classification does not depend on it, since it has not been modeled or verified as part of this feature's analysis.

---

## 4. Method

- **HARA (ISO 26262):** the hazard arises from a *malfunction* — a component or its software behaves incorrectly relative to specification. Classified via S/E/C -> ASIL.
- **SOTIF (ISO 21448):** the hazard arises with *no malfunction* — every component performs as specified, and the specification/performance is simply insufficient for some triggering condition. Not classified via ASIL.
- **Not safety-relevant:** no plausible path to human harm. Excluded from both.

---

## 5. Exhaustive Effects List

1. Cloud data reaching the Prior layer is corrupted or spoofed in transit or at rest.
    * Category: **HARA**
    * Rationale: Data reaching a safety-relevant function incorrect relative to spec — an external malfunction.

2. A defect in the Live-layer-population logic (block_diagram.md Section 3) causes Perception's output to be incorrectly represented in the fused grid — independent of whether Perception's own detection was correct.
    * Category: **HARA**
    * Rationale: A software defect in this feature's own logic relative to spec, distinct from #1 (external origin) and #3 (a specific sub-case of misinterpreting absence).

3. The degraded-input-handling logic (block_diagram.md Section 3) fails to correctly distinguish a missing/degraded layer from a confident observation — either direction: treating absence as a valid negative, or misreading a missing value as a valid positive.
    * Category: **HARA**
    * Rationale: A software defect in this feature's own logic relative to its stated specification (block_diagram.md Section 3), distinct from #2 in that the defect is specifically in how *absence* of input is handled, not in how present input is transcribed.

4. Perception misidentifies a visual artifact (shadow, tar patch, wet asphalt) as a pothole at high confidence — a "ghost pothole."
    * Category: SOTIF
    * Rationale: Perception performs exactly as designed; the algorithm's discrimination performance is simply insufficient in this triggering condition. No component failed.

5. Telemetry from the Pothole Observation Reporter, relayed by the Connectivity Manager, consumes bus bandwidth or ECU cycles, delaying other time-critical safety messages.
    * Category: **HARA**
    * Rationale: Systematic design fault — an unbounded or unprioritized transmission path is a specification/architecture defect.

6. A vehicle's Perception fails to detect an active severe pothole it drives over, and — since no physical-confirmation sensor exists in this design — no other onboard mechanism corrects this for that pass.
    * Category: SOTIF
    * Rationale: Perception operates within its designed performance envelope; the miss is a known category of sensor limitation, not a malfunction. Confidence that a hazard is genuinely present or absent accrues entirely from fleet-scale observation diversity over time — this feature includes no onboard physical-confirmation sensor. A single vehicle's miss on a single pass reduces the vehicle to baseline (pre-feature) risk for that pass, not an elevated one.

7. World Model Builder's fusion of the Live and Prior depth-grid patches exceeds its real-time budget under high load — many simultaneous candidates, a large cloud-advisory list, or large patch dimensions (N×M).
    * Category: **HARA**
    * Rationale: Timing/resource malfunction, contributing to World Model Builder's existing real-time-performance hazard class. This feature's grid-patch fusion compute is a genuine addition to that budget, not merely inherited.

8. Localization spatial drift in GNSS-degraded environments causes the Prior layer's grid patch to be placed at the wrong position within World Model Builder's ego-centric grid.
    * Category: SOTIF
    * Rationale: Environmental performance limitation of the localization sensor stack, not a malfunction. Already in scope of the existing Localization item's SOTIF case, independent of this feature. **This applies to the Prior layer specifically — converting the cloud's absolute-coordinate entry into a position within the vehicle's ego-relative grid requires an accurate ego position.** It does not apply to the Live layer in the same way: Perception's output arrives already ego-relative (a distance measurement), so no localization-dependent conversion is needed to place it in the grid. Localization only affects Live-layer-derived content much later, when the Reporter exports an absolute position for the cloud report — a minor, dead-end-bounded accuracy concern (see fmea.md), not a fusion-safety one.

9. Repeated pothole impacts increase suspension/tire/sensor wear and accelerate maintenance cost.
    * Category: NOT SAFETY-RELEVANT — excluded
    * Rationale: No plausible path to human harm.

---

## 6. Classification (items #1, #2, #3, #5, #7)

**1. Corrupted or spoofed cloud data reaches the Prior layer**
- Operational situation: High-speed highway, dense/adjacent traffic
- Vehicle-level effect: Unintended abrupt swerve or hard braking -> lane departure / rear-end collision
- S / E / C: S3 / E4 / C3
- ASIL: D
- Disposition: Defended by the Connectivity Manager's authentication function (Baseline Dependency D) at the point external data enters the vehicle. This feature's own Prior-layer-population logic (transcribing already-validated cloud data into the layer format) must still be developed at ASIL D, per World Model Builder's classification (Section 2) — it is not a separately classifiable lower-rigor task.

**2. Live-layer-population logic defect**
- Operational situation: High-speed highway, dense/adjacent traffic
- Vehicle-level effect: Unintended abrupt swerve or hard braking -> lane departure / rear-end collision
- S / E / C: S3 / E4 / C3
- ASIL: D
- Disposition: This logic executes inside World Model Builder's own processing without architectural isolation (Section 2) and must be developed and verified at ASIL D directly — there is no basis for classifying it separately at a lower rigor.

**3. Degraded-input-handling logic defect**
- Operational situation: High-speed highway, dense/adjacent traffic, concurrent with degraded Perception compute or unavailable/stale cloud data
- Vehicle-level effect: Unintended abrupt swerve or hard braking (if absence is misread as a positive observation) -> lane departure / rear-end collision. The opposite direction (a real hazard misread as absent) reduces to baseline risk, per item #6's reasoning — it does not independently warrant a separate S/E/C.
- S / E / C: S3 / E4 / C3
- ASIL: D
- Disposition: Same basis as #2 — this logic is integral to World Model Builder's own processing (Section 2) and must be developed and verified at ASIL D directly.

**5. Telemetry burst starves safety-critical bus**
- Operational situation: Dense urban / rough terrain, high-frequency reporting
- Vehicle-level effect: Delayed processing of time-critical messages -> late collision-avoidance execution
- S / E / C: S3 / E4 / C3
- ASIL: D
- Disposition: **INHERITED-BY-ARCHITECTURE** — pre-existing E/E bandwidth-budget hazard class; resolved via a bandwidth/priority allocation requirement, not a feature-specific Safety Goal. The Pothole Observation Reporter's own QM classification (Section 2) does not change this — a QM component's unbounded traffic can still starve an ASIL D bus, which is why this remains a distinct architecture-level requirement.

**7. World Model Builder real-time budget overrun**
- Operational situation: Dense urban, multiple concurrent dynamic obstacles and perceiver inputs; alternatively, a dense-pothole area or large patch dimensions increasing fusion compute
- Vehicle-level effect: Delayed world-model output
- S / E / C: S2 / E4 / C2
- ASIL: B
- Disposition: **INHERITED**, into World Model Builder's existing real-time-performance hazard class, resolved via its existing WCET/real-time budget requirement — this feature's grid-patch fusion compute is now part of that budget, not a separately-owned one. Under sustained budget pressure (including fail-operational or reduced-clock conditions), this feature requires an explicit shedding priority: the Prior (cloud) layer is dropped first, since it is comfort/proactive-avoidance content whose absence reduces to baseline risk (item #6's reasoning), before any core safety layer (obstacles, lane boundaries) is affected. This is the direct analogue, at World Model Builder's level, of Perception's own existing internal prioritization (block_diagram.md Section 1: pothole classification dropped first under Perception's compute pressure) — no equivalent statement currently exists for World Model Builder's fusion budget.

---

## Classification Summary

This feature does not have an independently classifiable QM item for its world-model-facing contribution. The Live-layer-population and degraded-input-handling logic execute as an integral part of World Model Builder's own processing, without architectural isolation from it — per ISO 26262 coexistence principles, this means that logic must be developed and verified at World Model Builder's existing ASIL D, not classified separately at a lower rigor.

The Pothole Observation Reporter remains independently classifiable at **QM**: it is architecturally separate from World Model Builder (reads its fused output, but its own logic and failures have no path back into any safety-critical chain — a genuine dead end, reporting only to the cloud).

Perception's pothole-candidate generation was never a candidate for separate QM classification; it has always been part of Perception's own existing, non-QM development.

Cloud-side logic (Map Update & Healing Engine, aggregation) is external to the vehicle and out of this HARA's scope by definition.

---

## 7. SOTIF Items (items #4, #6, #8) — not ASIL-classified here

- **#4 Ghost pothole:** a question of baseline Perception's discrimination performance, not this feature's.
- **#6 Sensor miss, no fleet corroboration yet:** whether baseline Perception is adequate for negative-obstacle detection is a real question, but belongs to the vehicle's baseline perception SOTIF case, not this feature.
- **#8 Localization drift (Prior layer only):** inherited from the existing Localization item's SOTIF case; not new to this feature.

---

## 8. Findings

1. This feature's world-model-facing logic (Live-layer population, degraded-input handling) must be developed and verified at ASIL D — it has no separately classifiable QM item, because it is integrated into World Model Builder without architectural isolation from World Model Builder's own core processing.
2. The Pothole Observation Reporter remains classifiable at QM — the only genuinely separable piece of this feature's onboard logic, being a dead end with no path back into the safety-relevant chain.
3. World Model Builder's real-time budget requires an explicit shedding priority under sustained pressure: Prior (cloud) layer first, before any core safety layer is affected — the analogue of Perception's own existing internal prioritization, previously unstated at World Model Builder's level.
4. Whether Perception should produce a separately-defined pothole-candidate output at all, versus road-surface content already being one semantic class within Perception's unified occupancy/semantic output (documented as the more current pattern, e.g. BEVFusion, already cited in block_diagram.md), is an open architectural question this document flags but does not resolve.
5. Localization drift affects the Prior layer's placement in World Model Builder's grid, not the Live layer's. The Live layer arrives already ego-relative and needs no localization-dependent conversion to be placed in the grid.

---

## 9. Traceability to Next Document

- **World Model Builder ASIL D development requirement** -> this feature's Live-layer-population and degraded-input-handling logic (block_diagram.md Section 3) must be developed and verified to ASIL D as an integral part of World Model Builder (addresses #1, #2, #3).
- **Connectivity Manager authentication/validation requirement (baseline)** -> confirm coverage of this feature's cloud data before it reaches the Prior layer (addresses #1).
- **Bandwidth allocation requirement** -> derive from existing E/E architecture budget (addresses #5).
- **World Model Builder real-time budget requirement, inclusive of grid-patch fusion compute and an explicit shedding priority (Prior layer first)** -> derive from existing WCET allocation, extended to state this feature's degradation order explicitly (addresses #7).
- **Pothole Observation Reporter QM development** -> threshold/debounce logic and report packaging, developed at QM rigor consistent with its dead-end architectural position (supports Classification Summary).
- **Open architectural question — unified vs. separate Perception output for road-surface content** -> resolve whether interface #1 should exist as a dedicated feed at all, or whether this feature should instead rely on Perception's already-unified occupancy output (Finding 6).