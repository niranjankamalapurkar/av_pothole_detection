## Scope: 
This document is limited to hazard identification and ASIL classification for **functional safety (ISO 26262)**. Items that trace to **SOTIF (ISO 21448)** are identified and separated out, but not classified with an ASIL — they require scenario-based residual-risk acceptance, which is a different process and out of scope for this document. Items that are pure vehicle/business impact (no path to human harm) are excluded from HARA entirely. Safety mechanisms and architectural allocation belong in the downstream Functional Safety Concept (FSC).

---

## 1. Purpose

The Pothole Detection feature introduces a new input vector into the vehicle's trajectory-planning item. This document determines:

1. Which existing item(s) this feature modifies.
2. Whether the feature introduces **functional-safety** hazardous events not already covered by an existing HARA (vs. SOTIF concerns, vs. no safety relevance at all).
3. ASIL classification for any new HARA hazardous events, derived from S/E/C — not asserted.

This is a **Delta HARA**, not a ground-up HARA: it assumes the baseline vehicle-level HARA (dynamic driving task, lane keeping, collision avoidance) already exists and is unchanged.

---

## 2. Item Definition (Delta)

Item: Path Planning & Trajectory Arbitration
    Status:          Existing item — MODIFIED (new input)
    ASIL (existing): D (unchanged)

Item: Vehicle Controls (steering/braking actuation)
    Status:          Existing item — UNAFFECTED
    ASIL (existing): D (unchanged)

Item: Perception, Sensor Fusion, Localization
    Status:          Existing items — unaffected in isolation; Sensor Fusion gains a new jerk input
    ASIL (existing): Unchanged

Item: Pothole Detection & Geotagging chain (new)
    Scope:           Sensor Fusion pothole logic, Geotagging & Verification Engine,
                      Connectivity Manager, Local Datalogger
    ASIL (existing): Under determination — see Section 6

**Why Path Planning, not Vehicle Controls:** per the interface spec, the pothole feed terminates at the Central Path Planner (interfaces #6, #8). Vehicle Controls only receives the Path Planner's final arbitrated command (interface #10) and has no visibility into pothole data — it is not an impacted item.

---

## 3. Assumptions Carried from Baseline HARA

- Baseline vehicle-level hazards (unintended lateral trajectory violation, unintended braking/acceleration) already have an existing Safety Goal, referred to here as **SG-01**.
- The baseline S/E/C ratings for those hazards do not change due to this feature — this is a system-level judgment inherited from the baseline item, not re-derived here.
- The Path Planner has a real-time trajectory arbitration and occupancy-grid validation function governing all candidate trajectory modifications regardless of source — inherited, not re-proven here.

---

## 4. Method

For every candidate effect, the first question is **not** "what's the ASIL" — it's **"what kind of thing is this?"**:

- **HARA (ISO 26262):** the hazard arises from a *malfunction* — a component, or the software/logic, behaves incorrectly relative to its specification (random HW fault, systematic SW defect, corrupted/malicious data reaching a function). Classified via S/E/C -> ASIL.
- **SOTIF (ISO 21448):** the hazard arises with **no malfunction** — every component does exactly what it was specified to do, and the specification/performance is simply insufficient for some triggering condition (sensor physics, environmental edge case, algorithmic limitation). Not classified via ASIL; addressed via scenario coverage and residual-risk acceptance criteria.
- **Not safety-relevant:** the effect has no plausible path to human harm (e.g., component wear, cost, downtime). Excluded from both.

---

## 5. Exhaustive Effects List

1. Cloud map payload is spoofed or corrupted in transit/storage, delivering fake severe-pothole   coordinates to the Path Planner.
    * Category:  **HARA**
    * Rationale: Data reaching a safety-relevant function is incorrect relative to spec — a malfunction, regardless of whether the root cause is malicious or a random fault.
2. Perception/fusion misidentifies a visual artifact (shadow, tar patch, wet asphalt) as a pothole at high confidence — a "ghost pothole."
    * Category:  SOTIF
    * Rationale: Perception performs exactly as designed; the algorithm's discrimination performance is simply insufficient in this triggering condition. No component failed.
3. Telemetry/datalogger transmission bursts consume bus bandwidth or ECU cycles, delaying other time-critical safety messages.
    * Category:  **HARA**
    * Rationale: Systematic design fault — an unbounded or unprioritized transmission path is a specification/architecture defect, not a performance-limitation-under-nominal-operation issue.
4. A vehicle fails to detect an active severe pothole it physically drives over (sensor miss — lens flare, straddling) and the system emits a CLEARED status.
    * Category:  SOTIF
    * Rationale: The perception system is operating within its designed performance envelope; the miss is a known category of sensor limitation, not a malfunction.
5. The cloud Verification/Healing Engine's confirmation logic contains a software defect that causes it to emit CLEARED status independent of any individual vehicle's detection performance (e.g., threshold/counting bug).
    * Category:  **HARA**
    * Rationale: This is a distinct causal path from #4 — here the logic itself is defective relative to its specification (e.g., an off-by-one confirmation-count bug). Same top-level vehicle effect as #4, different root-cause category, and therefore needs separate treatment.
6. Path Planner receives a rapid stream of conflicting constraint modifiers (pothole nudge + pedestrian + merge) and its trajectory computation exceeds its real-time loop budget.
    * Category:  **HARA**
    * Rationale: Timing/resource malfunction of an existing item; the pothole feed is simply one more contributor to an already-existing hazard class for the Path Planner.
7. Localization spatial drift in GNSS-degraded environments (urban canyon, tunnel) causes the Geotagging Engine to associate a real pothole with the wrong lane coordinates.
    * Category:  SOTIF
    * Rationale: Environmental performance limitation of the localization sensor stack, not a malfunction. Already in scope of the existing Localization item's SOTIF case, independent of this feature — not new.
8. Repeated pothole impacts increase suspension/tire/sensor wear and accelerate maintenance cost. (Carried over from problem_statement.md motivation — included here only to explicitly rule it out of HARA.)
    * Category:  NOT SAFETY-RELEVANT — excluded
    * Rationale: No plausible path to human harm. This is a fleet-maintenance/business-case consideration (correctly the domain of problem_statement.md), not a HARA input.

---

## 6. HARA — ISO 26262 Classification (items #1, #3, #5, #6)

1. Spoofed/corrupted cloud payload -> false severe-pothole coordinate
    * Operational situation: High-speed highway, dense/adjacent traffic
    * Vehicle-level effect: Unintended abrupt swerve or hard braking -> lane departure / rear-end collision
    * S / E / C: S3 / E4 / C3
    * ASIL: D
    * Disposition: **INHERITED**; requires an input-plausibility/validation requirement on the Path Planner, not a new Safety Goal.

3. Telemetry burst starves safety-critical bus
    * Operational situation: Dense urban / rough terrain, high-frequency logging
    * Vehicle-level effect: Delayed processing of time-critical messages (object tracking, brake requests) -> late collision-avoidance execution
    * S / E / C: S3 / E4 / C3
    * ASIL: D
    * Disposition: **INHERITED-BY-ARCHITECTURE** — pre-existing E/E bandwidth-budget hazard class; resolved via a bandwidth/priority allocation requirement, not a feature-specific Safety Goal.

5. Verification Engine logic defect emits erroneous CLEARED status independent of sensor performance
    * Operational situation: High-speed fleet operation over a location previously flagged as a severe, verified pothole
    * Vehicle-level effect: None above baseline. Per interfaces.md, the onboard reactive path (Perception + Jerk -> Sensor Fusion -> Geotagging & Verification Engine -> Path Planner, interface #6) runs continuously and is not gated by cloud/advisory status — interfaces #7-#9 feed only the proactive, look-ahead layer. A vehicle reaching a location the cloud incorrectly marked CLEARED still scans it with its own real-time sensors exactly as it would for any unmapped pothole, i.e. it falls back to pre-feature baseline behavior rather than an elevated-risk state.
    * S / E / C: Not applicable — no vehicle-level hazard above baseline to classify.
    * ASIL: None. QM — architectural requirement only.
    * Disposition: **NOT A NEW HAZARD**. A malfunction that returns the system to baseline risk (rather than exceeding it) does not warrant a new Safety Goal. It requires a **freedom-from-interference requirement** instead (Section 10): the advisory layer must never be able to gate or override the reactive layer. This is currently an assumption read from the interface spec and should be explicitly confirmed against the actual design.

6. Path Planner real-time budget overrun from concurrent constraints
    * Operational situation: Dense urban, multiple concurrent dynamic obstacles
    * Vehicle-level effect: Delayed actuation or fallback to emergency stop
    * S / E / C: S2 / E4 / C2
    * ASIL: B
    * Disposition: **INHERITED** — pre-existing real-time-performance hazard class for the Path Planner; resolved via the existing WCET/real-time budget requirement, not a new hazardous event specific to this feature.

**Final item classification — Pothole Detection feature: QM.**

Rationale: every hazard traced above either (a) is inherited into an already-classified receiving item via a requirement placed on that item — Path Planner input-plausibility for #1, E/E bandwidth allocation for #3, Path Planner WCET budget for #6 — or (b) does not exceed baseline risk and requires only a freedom-from-interference argument (#5), or (c) is a SOTIF concern, not an ISO 26262 malfunction (#2, #4, #7 — Section 8). No hazard above requires the Pothole Detection itself to inherit or decompose an ASIL. This QM classification is conditional on the freedom-from-interference requirement (Section 10) being verified against the actual design — if verification shows the advisory layer can influence the reactive layer, this classification must be revisited and would likely require ASIL decomposition from SG-01 onto the chain.

---

## 8. SOTIF Items (items #2, #4, #7) — not ASIL-classified here

These require scenario-based validation and residual-risk acceptance criteria (ISO 21448), not an S/E/C/ASIL table:

- **#2 Ghost pothole:** requires a defined, testable discrimination-performance target (false-positive rate under specified visual conditions) and scenario coverage in validation. Vehicle-level effect is the same unintended-maneuver hazard as HARA #1, so the *effect* is already gated by the Path Planner's arbitration (SG-01) — but the SOTIF question is about acceptable *frequency* of triggering that gate under nominal (non-malfunctioning) operation, which is a separate acceptance case.
- **#4 Single-vehicle sensor miss:** requires a defined detection-performance target (miss rate under specified conditions — lens flare, partial occlusion). This is a property of the baseline reactive perception stack's adequacy for pothole-class (static, low-contrast) obstacles generally — it is not specific to, or made worse by, the cloud CLEARED-status pathway (see #5 above: a wrong cloud status doesn't suppress the reactive path). It should be tracked as a SOTIF validation target for the perception system, not folded into this feature's safety case as if the feature created the risk.
- **#7 Localization drift:** inherited from the existing Localization item's SOTIF case; not new to this feature, flagged only so it isn't silently dropped.

---

## 9. Findings

1. **This feature introduces zero new ASIL-rated Safety Goals.**
2. **One QM-level freedom-from-interference requirement is needed**, not a Safety Goal: the advisory (cloud) layer must never be able to gate or override the reactive (real-time onboard) layer. This should be explicitly verified against the real design, not assumed from the interface spec.
3. **The Pothole Detection feature is classified QM**, conditional on that freedom-from-interference requirement being verified against the actual design (see Section 6 rationale and Section 10).

---

## 10. Traceability to Next Document

- **SG-01 (inherited, ASIL D)** -> derive a Path Planner input-plausibility requirement for untrusted pothole payloads (addresses #1).
- **Bandwidth allocation requirement** -> derive from existing E/E architecture budget (addresses #3).
- **Real-time budget requirement** -> derive from existing Path Planner WCET allocation (addresses #6).
- **Freedom-from-interference requirement (QM, architectural)** -> the cloud/advisory status path must never gate, disable, or override the real-time reactive detection path, under any condition including erroneous or absent cloud data (addresses #5; also bounds the residual risk from #4).
- **SOTIF validation plan (separate work product, not this document)** -> define acceptance targets and scenario coverage for perception ghost-detection (#2), sensor miss-rate (#4), and localization drift (#7).