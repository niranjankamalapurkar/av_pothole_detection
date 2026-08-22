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

* Item: World Model Builder (baseline) 
    1. Status: Existing item — MODIFIED (new input: this feature's pothole entries)
    2. ASIL (existing): Assumed D, inherited — feeds Path Planner's safety-critical decisions; not independently re-derived in this document, the same treatment this document gives Path Planner itself.

* Item: Path Planning & Trajectory Arbitration
    1. Status: Existing item — UNAFFECTED DIRECTLY. Consumes the World Model Builder's output, which may now include pothole-derived entries, but has no interface with this feature at all and no special-case handling for it as a data source — every world-model entry is treated uniformly.
    2. ASIL (existing): D (unchanged)

* Item: Vehicle Controls (steering/braking actuation)
    1. Status: Existing item — UNAFFECTED
    2. ASIL (existing): D (unchanged)

* Item: Perception, Localization
    1. Status: Existing items. Perception gains an internal capability (pothole-candidate classification as one object class among others) but remains a single existing item, not decomposed into a separate one. Localization unaffected in isolation.
    2. ASIL (existing): Unchanged

* Item: Connectivity Manager (baseline) 
    1. Status: Existing item. Assumed to include a dedicated security/validation function (authentication, checksum, format checks) applied to all incoming cloud data before forwarding to any downstream consumer — baseline architecture, since Connectivity Manager almost certainly serves cloud-connected vehicle features beyond this one. This feature depends on that function but does not define it.
    2. ASIL (existing): Not independently derived here.

* Item: Pothole Cloud-Overlay Engine & Verification Engine (new) 
    1. Scope: As defined in block_diagram.md Sections 3-4. Replaces the "Sensor Fusion pothole logic, Geotagging & Verification Engine" scope from earlier architecture versions — those components no longer exist in this architecture.
    2. ASIL (existing): Under determination — see Section 6

**Why World Model Builder, not Path Planner directly:** per the current interface spec, this feature's pothole data terminates at the World Model Builder (interfaces.md #6) — a baseline item this feature feeds but does not define. Path Planner has no interface with this feature at all; it receives only the World Model Builder's already-merged output, structurally indistinguishable from any other perceiver's contribution.

---

## 3. Assumptions Carried from Baseline HARA

- Baseline vehicle-level hazards (unintended lateral trajectory violation, unintended braking/acceleration) already have an existing Safety Goal, referred to here as **SG-01**.
- The baseline S/E/C ratings for those hazards do not change due to this feature — this is a system-level judgment inherited from the baseline item, not re-derived here.
- Path Planner's real-time trajectory arbitration and occupancy-grid validation function governs all candidate trajectory modifications regardless of source — inherited, not re-proven here. This document does not assert what form that arbitration or bounding takes (magnitude ceiling, maneuver-type restriction, or otherwise) — that is Path Planner's own pre-defined, inherited logic.
- The World Model Builder is assumed to perform its own baseline input validation/plausibility checks on all perceiver inputs, including this feature's, before constructing the unified world model — inherited, not re-derived here. **This assumption is one of the conditions this document's QM classification depends on being verified (Section 6).**
- The Connectivity Manager is assumed to authenticate and validate all incoming cloud data (checksum, anti-spoofing, format checks) at the point it receives it from the network, before forwarding to any consumer — inherited baseline architecture, not re-derived here.

---

## 4. Method

For every candidate effect, the first question is **not** "what's the ASIL" — it's **"what kind of thing is this?"**:

- **HARA (ISO 26262):** the hazard arises from a *malfunction* — a component, or the software/logic, behaves incorrectly relative to its specification (random HW fault, systematic SW defect, corrupted/malicious data reaching a function). Classified via S/E/C -> ASIL.
- **SOTIF (ISO 21448):** the hazard arises with **no malfunction** — every component does exactly what it was specified to do, and the specification/performance is simply insufficient for some triggering condition (sensor physics, environmental edge case, algorithmic limitation). Not classified via ASIL; addressed via scenario coverage and residual-risk acceptance criteria.
- **Not safety-relevant:** the effect has no plausible path to human harm (e.g., component wear, cost, downtime). Excluded from both.

---

## 5. Exhaustive Effects List

1. Cloud data is spoofed, corrupted, or malformed, and reaches the Pothole Cloud-Overlay Engine as if valid, or the Overlay Engine's internal processing of that data corrupts, blocks, or delays its output for a genuine live detection.
    * Category: **HARA**
    * Rationale: Data reaching a safety-relevant function incorrect relative to spec is a malfunction, regardless of origin.
2. Perception misidentifies a visual artifact (shadow, tar patch, wet asphalt) as a pothole at high confidence — a "ghost pothole."
    * Category: SOTIF
    * Rationale: Perception performs exactly as designed; the algorithm's discrimination performance is simply insufficient in this triggering condition. No component failed.
3. Telemetry transmission bursts (Connectivity Manager) consume bus bandwidth or ECU cycles, delaying other time-critical safety messages.
    * Category: **HARA**
    * Rationale: Systematic design fault — an unbounded or unprioritized transmission path is a specification/architecture defect.
4. A vehicle fails to detect an active severe pothole it physically drives over (Perception miss — lens flare, straddling), and no jerk confirmation occurs either, causing Verification Engine to report CLEARED.
    * Category: SOTIF
    * Rationale: Perception and Jerk are both operating within their designed performance envelopes; the miss is a known category of sensor limitation, not a malfunction. Per the corrected Verification Engine logic (Section 6, item 5), this no longer produces an erroneous CLEARED report — it produces no report at all, which is the correct behavior for an ambiguous non-confirmation.
5. Onboard logic defect causes an erroneous CLEARED report — the Overlay Engine's CLOUD_CLEAR determination logic is defective (e.g., misreading perception_health_state, or a transient sensor issue not reflected in that state), or the cloud's own Healing Engine has a confirmation-logic defect, or Verification Engine's entry_type-plus-jerk mapping logic is defective.
    * Category: **HARA**
    * Rationale: Distinct from #4 (a performance limitation) — this is a logic defect relative to spec, wherever it originates. Narrower in scope than before this feature's CLOUD_CLEAR redesign: the previous, broader risk (any absence of jerk producing an erroneous CLEARED) is closed off entirely; what remains is specifically a defect in the CLOUD_CLEAR determination itself.
6. World Model Builder or Path Planner receives a rapid stream of conflicting or high-frequency inputs and exceeds its real-time loop budget.
    * Category: **HARA**
    * Rationale: Timing/resource malfunction of existing items for World Model Builder and Path Planner; a new but structurally identical hazard class for the Overlay Engine, whose matching logic scales with the size of the cloud-advisory list and the number of simultaneous live candidates — a genuine compute cost this feature introduces, not merely inherits.
7. Localization spatial drift in GNSS-degraded environments causes the Overlay Engine to mismatch a live detection against the wrong cloud entry, to mis-position a CLOUD_SUBSTITUTED entry, or to incorrectly confirm CLOUD_CLEAR at the wrong location.
    * Category: SOTIF
    * Rationale: Environmental performance limitation of the localization sensor stack, not a malfunction. Already in scope of the existing Localization item's SOTIF case, independent of this feature.
8. Repeated pothole impacts increase suspension/tire/sensor wear and accelerate maintenance cost.
    * Category: NOT SAFETY-RELEVANT — excluded
    * Rationale: No plausible path to human harm.
9. A downstream consumer (World Model Builder or Verification Engine) reads a field left at its default value for the current entry_type as if it were real data — e.g., distance_to_pothole_m defaulting to 0 for a CLOUD_SUBSTITUTED entry, misread as "hazard immediately at the vehicle" rather than "not applicable."
    * Category: **HARA**
    * Rationale: A specification/implementation defect — ambiguous sentinel values, or a consumer that does not gate field access on entry_type first — causing a component to act on default-as-if-valid data.


---

## 6. HARA — ISO 26262 Classification (items #1, #3, #5, #6, #9)

1. Corrupted or malformed pothole entry reaches the World Model Builder (external or internal cross-branch origin)
    * Operational situation: High-speed highway, dense/adjacent traffic
    * Vehicle-level effect: Unintended abrupt swerve or hard braking -> lane departure / rear-end collision
    * S / E / C: S3 / E4 / C3
    * ASIL: D
    * Disposition: **INHERITED.** The Connectivity Manager is the vehicle's network-facing boundary for cloud data — this is where authentication and anti-spoofing validation belongs, applied once, centrally, to all incoming cloud data before it reaches any consumer (Section 3). The Pothole Cloud-Overlay Engine relies on this upstream validation rather than re-implementing security checks itself; duplicating security logic across every cloud-consuming feature would be worse architecture, not better. World Model Builder's and Path Planner's existing plausibility/occupancy-grid checks remain a secondary backstop. Separately, the Overlay Engine's internal processing must isolate its cloud-processing branch from its live-detection branch, so a defect on one cannot delay, block, or corrupt the other — this is a distinct requirement from input authenticity, and it is this feature's own responsibility, since it concerns the Overlay Engine's internal design, not the Connectivity Manager's.

3. Telemetry burst starves safety-critical bus
    * Operational situation: Dense urban / rough terrain, high-frequency logging
    * Vehicle-level effect: Delayed processing of time-critical messages (object tracking, brake requests) -> late collision-avoidance execution
    * S / E / C: S3 / E4 / C3
    * ASIL: D
    * Disposition: **INHERITED-BY-ARCHITECTURE** — pre-existing E/E bandwidth-budget hazard class; resolved via a bandwidth/priority allocation requirement, not a feature-specific Safety Goal.

5. Onboard or cloud-side logic defect causes an erroneous CLEARED report
    * Operational situation: High-speed fleet operation over a location previously flagged as a severe, verified pothole
    * Vehicle-level effect: None above baseline. Verification Engine's report only reaches the cloud database via the Connectivity Manager (interface #8) — it has no path to the World Model Builder or Path Planner. A vehicle reaching a location the cloud incorrectly marked CLEARED still has its own Perception scan that location fresh; if a live detection occurs, Overlay Engine forwards it (as LIVE_ONLY if unmatched, since a CLEARED location would no longer be on the cloud's advisory list), regardless of the erroneous cloud status.
    * S / E / C: Not applicable — no vehicle-level hazard above baseline to classify.
    * ASIL: None. QM.
    * Disposition: **NOT A NEW HAZARD.** A malfunction that returns the system to baseline risk does not warrant a new Safety Goal. Requires confirmation that Verification Engine's output path is genuinely a dead end with no route back into the world-model or Path-Planner-facing chain. Note this item's scope narrowed materially in this revision: an erroneous CLEARED can now only originate from a genuine logic defect in the CLOUD_CLEAR determination, the cloud's Healing Engine, or the entry_type-to-status mapping — not from the mere absence of jerk confirmation, which previously could trigger CLEARED for any entry_type and specifically could not distinguish "not there" from "successfully avoided." That broader risk is now closed by design (block_diagram.md Section 4), not merely bounded by this disposition.


6. World Model Builder / Path Planner / Pothole Cloud-Overlay Engine real-time budget overrun
    * Operational situation: Dense urban, multiple concurrent dynamic obstacles and perceiver inputs; alternatively, a dense-pothole area producing a large cloud-advisory list for the Overlay Engine to match against
    * Vehicle-level effect: Delayed actuation or fallback to emergency stop
    * S / E / C: S2 / E4 / C2
    * ASIL: B
    * Disposition: **INHERITED** for World Model Builder and Path Planner — pre-existing real-time-performance hazard class for both, resolved via their existing WCET/real-time budget requirements. **NEW, this feature's own responsibility** for the Pothole Cloud-Overlay Engine — its matching-logic compute cost scales with cloud-advisory list size and the number of simultaneous live candidates, and needs its own defined real-time budget and worst-case complexity bound; not previously named as a distinct concern.

9. Sentinel field value misread as real data by a downstream consumer
    * Operational situation: Any situation where a CLOUD_SUBSTITUTED or CLOUD_CLEAR entry (or any entry_type with sentinel-valued fields) reaches a consumer that does not gate on entry_type first
    * Vehicle-level effect: Same class as #1 — a spuriously "immediate" or otherwise incorrect hazard signal reaching Path Planner via the World Model Builder
    * S / E / C: S3 / E4 / C3
    * ASIL: D
    * Disposition: **INHERITED** — same primary defense as #1 (World Model Builder / Path Planner plausibility validation). Requires an explicit specification-level mitigation: sentinel values must be unambiguous (never a valid-looking number such as 0), and every consumer must gate on entry_type before reading any other field. interfaces.md #6 now specifies field population exhaustively per entry_type (previously illustrative only), closing part of this gap; the exact sentinel representation itself remains an open specification item.

**Final item classification — Pothole Cloud-Overlay Engine & Verification Engine: QM.**
 
Rationale: every hazard traced above either (a) is inherited into an already-classified receiving item via a requirement placed on that item — Connectivity Manager's authentication/validation function and World Model Builder / Path Planner plausibility validation for #1 and #9, E/E bandwidth allocation for #3, existing WCET budgets for #6's inherited portion — or (b) does not exceed baseline risk (#5, since Verification Engine's output has no path back into the safety-relevant chain, and its scope has narrowed to genuine logic defects only), or (c) is a SOTIF concern, not a malfunction (#2, #4, #7). No hazard above requires this feature's components to inherit or decompose an ASIL. This QM classification is conditional on **four** things being verified against the actual design — corrected in this revision from three, which omitted the World Model Builder's plausibility validation despite item #1's disposition explicitly relying on it: (1) Connectivity Manager's authentication/validation function genuinely exists and covers this feature's data; (2) Verification Engine's report path is genuinely a dead end; (3) the Overlay Engine's internal isolation between its cloud and live branches; (4) the World Model Builder's plausibility/occupancy-grid validation genuinely exists and applies uniformly to pothole-derived entries.

---

## 7. SOTIF Items (items #2, #4, #7) — not ASIL-classified here

These are performance-limitation (non-malfunction) items. None require a dedicated, feature-specific SOTIF validation effort — all three belong to existing baseline SOTIF cases this feature inherits rather than creates.
 
**Flagged, not feature-specific — no dedicated SOTIF work product needed here:**
- **#2 Ghost pothole:** on reconsideration, this was previously classified as requiring dedicated SOTIF validation, inconsistent with how #4 (the false-negative counterpart) was already correctly classified. A single vehicle's Perception misclassifying a visual artifact at high confidence is a question of baseline Perception's discrimination performance — this feature does not touch Perception's classification algorithm or confidence thresholds, and has no more claim to owning this SOTIF case than it does to owning #4's. Reclassified alongside #4 and #7.
- **#4 Single-vehicle sensor miss:** camera, LiDAR, and Jerk would all have to miss the same pothole simultaneously — jerk is not subject to the same visual-artifact triggering conditions as #2, since it only fires on physical contact. The main scenario defeating all three is geometric (wheels straddling the pothole entirely). Whether baseline Perception is adequate for negative-obstacle detection generally is a real question, but it belongs to the vehicle's baseline perception SOTIF case, not this feature.
- **#7 Localization drift:** inherited from the existing Localization item's SOTIF case; not new to this feature.

**What is this feature's own responsibility, distinct from the SOTIF question above:** a single vehicle's ghost detection is self-limiting — Verification Engine reports it, Path Planner's own bounded response absorbs it, done. What this feature *can* do, that a single vehicle's Perception limitation cannot on its own, is turn a rare, private hallucination into a persistent, fleet-wide artifact — because per block_diagram.md Section 7, a single vehicle report is currently sufficient to add a cloud entry, while clearing one requires multiple independent CLOUD_CLEAR confirmations. That asymmetry, and the cloud aggregation logic that does or doesn't guard against it, is this feature's own architecture, not Perception's. This is not a new item — it is folded into the aggregation-logic gap already identified in fmea.md Finding 9, now covering both the false-clearing risk (originally flagged) and false-adding/persistence risk (this reconsideration), since both stem from the same undefined aggregation logic.

---

## 8. Findings

1. This feature introduces zero new ASIL-rated Safety Goals.
2. Four requirements are needed, none of them a Safety Goal: 
    (a) Connectivity Manager's authentication/anti-spoofing validation must genuinely cover this feature's cloud data; 
    (b) the Pothole Cloud-Overlay Engine's cloud-processing branch must be internally isolated from its live-detection branch;
    (c) the single master-message design needs unambiguous sentinel values and strict entry_type-gating in every consumer; 
    (d) the World Model Builder's plausibility validation must genuinely exist and apply to pothole-derived entries — previously relied on in item #1's disposition but omitted from the classification's own conditionality list, corrected in this revision.
3. This feature's impacted item is the World Model Builder, not Path Planner — Path Planner has no interface with this feature at all, and this document does not assert what form Path Planner's own response-bounding logic takes.
4. The Pothole Cloud-Overlay Engine and Verification Engine are classified QM, conditional on the four requirements in Finding 2 being verified against the actual design.
5. The Overlay Engine's own real-time budget (item #6) is a genuinely new concern this feature introduces, not merely inherits — its matching-logic compute scales with cloud-advisory list size and simultaneous live-candidate count.
6. Verification Engine's erroneous-CLEARED risk (item #5) narrowed substantially by introducing CLOUD_CLEAR as a distinct entry_type: only a genuine logic defect in that determination can now produce an erroneous CLEARED — jerk absence alone, for any other entry_type, no longer can.
7. This feature owns no dedicated SOTIF work product. Ghost pothole (#2) was previously classified as requiring one, inconsistent with #4 and #7's correct classification as baseline Perception/Localization concerns — corrected in this revision. This feature's own responsibility regarding false-positive detections is narrower: not Perception's discrimination accuracy, but whether the cloud aggregation logic lets a single or correlated false report propagate to the fleet unchecked, now folded into the Healing Engine aggregation-logic requirement (Section 9).

---

## 9. Traceability to Next Document

- **Connectivity Manager authentication/validation requirement (baseline, ASIL not independently derived)** -> confirm the Connectivity Manager's existing security function covers this feature's cloud data before it reaches the Overlay Engine (addresses #1's external-origin case).
- **Overlay Engine internal freedom-from-interference requirement (QM, architectural)** -> the cloud-processing branch and the live-detection branch within the Pothole Cloud-Overlay Engine must be isolated such that a defect on one cannot delay, block, or corrupt the other's output (addresses #1's internal-origin case).
- **World Model Builder / Path Planner input-plausibility validation (inherited, ASIL D — secondary backstop)** -> any world-model entry, regardless of source, is subject to existing plausibility/occupancy-grid checks before influencing a trajectory decision (addresses #1, #9).
- **Bandwidth allocation requirement** -> derive from existing E/E architecture budget (addresses #3).
- **Real-time budget requirement — World Model Builder and Path Planner** -> derive from existing WCET allocations (addresses #6, inherited portion).
- **Real-time budget requirement — Pothole Cloud-Overlay Engine (new)** -> define a worst-case matching-logic complexity bound as a function of cloud-advisory list size and simultaneous live-candidate count, and a corresponding real-time budget (addresses #6, new portion).
- **CLOUD_CLEAR determination correctness requirement (QM, new)** -> the Overlay Engine's logic for confirming perception_health_state = NOMINAL and an active non-detection at a cloud-expected location must be verified against the actual design, since this is now the sole basis for a CLEARED report (addresses #5).
- **Verification Engine dead-end confirmation (QM, architectural)** -> explicitly verify Verification Engine's output has no route back into the world-model or Path-Planner-facing chain (addresses #5).
- **Sentinel-value and entry_type-gating specification (QM)** -> interfaces.md #6 now specifies field population exhaustively per entry_type; the exact sentinel representation (never a valid-looking number) remains to be defined (addresses #9).
- **SOTIF: no dedicated work product owned by this feature.** Ghost pothole (#2), sensor miss-rate (#4), and localization drift (#7) all belong to baseline Perception/Localization SOTIF cases, not this feature — reclassified in this revision (#2 was previously, inconsistently, treated as requiring dedicated validation). This feature's actual responsibility is narrower and already captured under the Healing Engine aggregation-logic requirement below.
- **Healing Engine aggregation-logic requirement (QM, new)** -> the cloud's threshold, time window, and independence criteria for both adding and clearing a pothole entry from a vehicle report must be explicitly defined — not just "N reports," but N reports demonstrated to be independent rather than correlated. This single requirement now covers two distinct risks: correlated false CLEARED reports (multiple vehicles all avoiding the same real pothole, addresses hara.md item #5's residual concern) and correlated or single-report false ADDED entries (a ghost detection, or the same visual artifact independently hallucinated by multiple vehicles under similar conditions, persisting in the shared database). What counts as "correlated" — matching environmental conditions, time-of-day, lighting, or something else — is not yet defined and needs to be resolved explicitly in the functional requirements document, not assumed here.