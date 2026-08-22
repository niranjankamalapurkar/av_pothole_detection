## Scope

This is an independent, bottom-up Failure Mode and Effects Analysis (FMEA) of the Pothole Detection & Cloud-Overlay feature, rebuilt against the current architecture (block_diagram.md, interfaces.md) rather than patched onto an FMEA built against an earlier pipeline shape. Component names, interfaces, and even the overall data-flow topology have changed substantially since the previous FMEA revision — Sensor Fusion, the Enablement Gate, Geotagger, and Output Arbitrator no longer exist in any form.

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
    Local effect:  No pothole candidates produced this cycle.
    System effect: Overlay Engine's "no usable live detection" branch triggers automatically on an empty candidate array — no separate fail-safe mechanism is needed.
    Coverage:      COVERED — block_diagram.md Section 3 (Overlay Engine's branch logic)

**2. Failure mode: Visual misclassification (ghost pothole)**
    Category:      Erroneous output
    Coverage:      COVERED — hara.md item #2

**3. Failure mode: Miss on a real pothole**
    Category:      Erroneous output (false negative)
    Coverage:      COVERED — hara.md item #4

**4. Failure mode: Compute-resource contention with primary object detection**
    Category:      Delayed output / partial loss of function
    Local effect:  Pothole classification competes with primary object detection for shared compute.
    System effect: Perception's own internal prioritization drops pothole classification first, per block_diagram.md Section 1. Whether this behavior is actually implemented (vs. merely documented as an assumption) is not independently verifiable from the architecture documents alone.
    Coverage:      COVERED — block_diagram.md Section 1

**5. Failure mode: perception_health_state was described but not carried on any defined interface — RESOLVED**
    Category:      Documentation/specification gap, not a component failure per se
    Local effect:  Previously, block_diagram.md Section 1 described perception_health_state as a real signal without interfaces.md #1's payload carrying it.
    System effect: The timestamp enables latency measurement between Perception producing a cycle and Overlay Engine consuming it. No further gap here.
    Coverage:      COVERED — interfaces.md #1

---

## Jerk/IMU Sensor

**1. Failure mode: Complete loss of function**
    Category:      Loss of function
    Local effect:  No jerk confirmation available to Verification Engine.
    System effect: sensor_health_state = FAULTED is treated as unknown, not zero-jerk (interfaces.md #2). Under the corrected entry_type-plus-jerk logic (block_diagram.md Section 4), an unknown jerk reading for any entry_type other than CLOUD_CLEAR simply produces no report — it was never sufficient to produce CLEARED and still is not sufficient to produce NEW or ACTIVE.
    Coverage:      COVERED — interfaces.md #2

**2. Failure mode: Vehicle speed context no longer reaches Verification Engine - RESOLVED**
    Category:      Design gap, not a sensor failure mode
    Local effect:  Previously, no interface carried vehicle speed to Verification Engine. 
    System effect: Now carried explicitly (interfaces.md #5) — used for jerk-severity calibration and, combined with a live entry's distance and timestamp, arrival-time estimation. 
    Coverage: COVERED — interfaces.md #5, block_diagram.md Section 4

---

## Localization

**1. Failure mode: Drift or complete loss of position fix**
    Category:      Erroneous output (drift) / Loss of function (total loss)
    Local effect:  Overlay Engine's matching (live-to-cloud correlation), positioning (CLOUD_SUBSTITUTED placement), or CLOUD_CLEAR confirmation (is the vehicle actually at the cloud-expected location) becomes unreliable; Verification Engine's live position tracking (interfaces.md #4), used to correlate jerk events against an entry's stored lat/lon, becomes unreliable. 
    System effect: localization_confidence (interfaces.md #3, #4) lets both consumers detect this and discount/suppress accordingly. CLOUD_CLEAR is a new consumer of this signal not present in earlier revisions — a drifted position could cause a false CLOUD_CLEAR (vehicle believes it's at a cloud-expected location and finds nothing, but is actually elsewhere) or a missed one. Degraded Localization also weakens Verification Engine's correlation for CLOUD_SUBSTITUTED/CLOUD_CLEAR entries specifically, since the live position check is their only correlation mechanism (no live distance measurement exists to fall back on kinematic timing). C
    overage: COVERED — hara.md item #7, interfaces.md #3/#4

---

## Pothole Cloud-Overlay Engine

**1. Failure mode: Cloud-branch processing corrupts, delays, or blocks the live-detection branch**
    Category:      Erroneous output / Delayed output (malfunction, HARA category)
    Local effect:  A defect or malformed cloud payload being processed on the cloud-advisory branch affects the live-detection branch's output for the same cycle. 
    System effect: This engine is the correct, dedicated owner of the reconciliation job. Distinct from input authenticity (Connectivity Manager's responsibility, interfaces.md Baseline Dependency D) — this concern is about isolating the two branches from each other even for data that has already passed upstream validation but still triggers a defect during processing. 
    Coverage: Input authenticity is Connectivity Manager's responsibility (Baseline Dependency D). Internal branch isolation NOT COVERED — see hara.md item #1 and Finding 2

**2. Failure mode: Live-to-cloud matching logic has an undefined tolerance/algorithm**
    Category:      Design gap
    Local effect:  block_diagram.md Section 3 says the Overlay Engine matches a live detection "against the cloud-advisory list by position," with no defined tolerance or matching criteria, and no defined behavior for multiple simultaneous live candidates against multiple cloud entries (an assignment problem, not a single lookup). 
    System effect: Too loose a tolerance risks a false match. Too tight a tolerance risks a false non-match — duplicate world-model entries for the same physical pothole. With multiple simultaneous candidates, an undefined assignment algorithm risks cross-matching (candidate A matched to candidate B's cloud entry). 
    Coverage: NOT COVERED — see Finding 3

**3. Failure mode: Sentinel values in the single master message misread as real data**
    Category:      Erroneous output (specification defect)
    Local effect:  A field left at its sentinel for the current entry_type is structurally indistinguishable from a genuine value unless the consumer gates on entry_type first.
    System effect: interfaces.md #6 now specifies field population exhaustively per entry_type (including the newly added CLOUD_CLEAR), closing part of this gap. The exact sentinel representation (guaranteed never to look like a valid value) remains unspecified.
    Coverage:      Field-population table now exhaustive — interfaces.md #6. Exact sentinel representation NOT COVERED — see hara.md item #9 and Finding 4
 
**4. Failure mode: CLOUD_CLEAR determination is incorrect due to a flaw in reading perception_health_state or position confirmation**
    Category:      Erroneous output (malfunction, HARA category)
    Local effect:  The Overlay Engine incorrectly concludes perception_health_state = NOMINAL when it is not, or incorrectly concludes the vehicle is at a cloud-expected location when it is not, and produces a false CLOUD_CLEAR.
    System effect: Since CLOUD_CLEAR is now the sole basis for a CLEARED report (block_diagram.md Section 4), a defect here has a more concentrated effect than under the previous design, where erroneous CLEARED could originate from several looser conditions. This concentration is a deliberate trade-off — it closes off the broader, harder-to-defend risk (jerk absence alone) in exchange for one narrower, more precisely defined determination that needs to be correct.
    Coverage:      NOT COVERED — see hara.md item #5 and Finding 5
 
**5. Failure mode: Own real-time budget undefined**
    Category:      Delayed output
    Local effect:  Matching logic's compute cost scales with cloud-advisory list size and the number of simultaneous live candidates.
    System effect: No worst-case complexity bound or real-time budget is defined for this engine specifically, distinct from World Model Builder's and Path Planner's budgets.
    Coverage:      NOT COVERED — see hara.md item #6 and Finding 6
 
**6. Failure mode: Shared-memory broadcast (interface #6) has no stated synchronization mechanism**
    Category:      Erroneous output (malfunction — torn read)
    Local effect:  One writer (Overlay Engine), two readers (World Model Builder, Verification Engine), no stated versioning, atomic-write guarantee, or lock.
    System effect: A reader could observe a partially-written message — e.g., an updated entry_type alongside stale field values from the previous cycle — if it reads mid-write. This is an implementation-level concern that belongs in functional/software requirements, not an architecture change.
    Coverage:      NOT COVERED — see Finding 7

---

## Verification Engine
 
**1. Failure mode: entry_type-plus-jerk mapping logic incorrectly determines NEW/ACTIVE/CLEARED**
    Category:      Erroneous output (malfunction, HARA category)
    Coverage:      COVERED — hara.md item #5
 
**2. Failure mode: Reported location could be imprecise if computed differently across entry types**
    Category:      Erroneous output (minor)
    Local effect:  For LIVE_ONLY/LIVE_ENRICHED, lat/lon now arrives directly on the entry (interfaces.md #7), computed by the Overlay Engine at detection time — no longer a separate lookup by Verification Engine, closing the previous staleness concern about a mismatched later Localization read.
    System effect: A minor accuracy concern for the cloud database (position reflects detection time, not confirmation time, if the vehicle moved meaningfully between the two), not a real-time vehicle-level hazard, since this component's output never reaches Path Planner or the world model (hara.md item #5's disposition).
    Coverage:      COVERED — interfaces.md #7. Minor residual accuracy note only, no action item.
 
**3. Failure mode: Multiple simultaneous in-flight entries could be cross-correlated to the wrong jerk event**
    Category:      Erroneous output (malfunction, HARA category)
    Local effect:  If more than one pothole entry is awaiting confirmation at the same time (e.g., two candidates close together), Verification Engine's correlation logic (position check plus, where available, kinematic timing) needs to correctly assign an incoming jerk event to the right entry, not just find *a* plausible match.
    System effect: A cross-correlation error would misreport status for the wrong pothole_id — e.g., confirming entry A as ACTIVE using jerk evidence that actually belongs to entry B, while entry B itself goes unconfirmed. This is the Verification-Engine-side counterpart to Finding 3 (the Overlay Engine's own multi-candidate matching problem) — same class of issue, different component.
    Coverage:      NOT COVERED — see Finding 9
 
---
 
## Connectivity Manager / Local Datalogger / Cloud
 
**1. Failure mode: Network-down fallback fails; intermittent/partial/corrupted data**
    Category:      Loss of function / Intermittent operation
    System effect: Bounded by the same reasoning as before — Verification Engine's output has no path back into the world model or Path Planner, so cloud-side failures cannot elevate real-time risk. Intermittent/partial data is rejected outright per interfaces.md #5.
    Coverage:      COVERED — hara.md item #5, interfaces.md #5
 
**2. Failure mode: Cloud-advisory list staleness has no defined threshold**
    Category:      Erroneous output
    Local effect:  interfaces.md #5 now carries a timestamp on the advisory list, but no staleness threshold is defined for how old is too old to act on.
    System effect: Overlay Engine could match, substitute, or confirm CLOUD_CLEAR against data that no longer reflects current road conditions.
    Coverage:      NOT COVERED — see Finding 1
 
**3. Failure mode: Healing Engine's multi-report aggregation logic is unspecified — for both adding and clearing**
    Category:      Design gap
    Local effect:  block_diagram.md Section 7 states multiple vehicles reporting CLEARED clears a pothole, and a single vehicle report is sufficient to add one — but the threshold, time window, and whether reports are checked for independence are undefined in both directions.
    System effect: If aggregation treats correlated reports (many vehicles running the same avoidance logic against the same real pothole, or many vehicles independently hallucinating the same visual artifact) the same as independent confirmations, the corroboration safeguard is weaker than it appears in both directions — false clearing of real potholes, and persistence of ghost ones. The CLOUD_CLEAR redesign (this revision) reduces how often a correlated false clear could occur, since CLOUD_CLEAR requires an active, healthy non-detection rather than a passive non-confirmation — but it does not address the add-side risk at all, and a single unverified report currently reaches the fleet-wide advisory list unchecked.
    Coverage:      NOT COVERED — see Finding 8
 
---
 
## World Model Builder / Path Planner / Vehicle Controls (baseline — largely out of scope)
 
**Note:** this feature has no direct interface with Path Planner or Vehicle Controls. The only remaining feature-relevant question for these baseline items is whether they correctly apply their existing validation/budget behavior uniformly to pothole-derived world-model entries, and whether the World Model Builder correctly treats CLOUD_CLEAR as a no-op — both captured as inherited assumptions in hara.md Section 3 rather than re-analyzed here.
 
---
 
## Findings Summary
 
**Finding 1 —** Cloud-advisory list staleness threshold undefined
 
interfaces.md #5 now carries a timestamp, but nothing states how old is too old to act on.
 
Action: define a staleness threshold for the cloud-advisory list, past which the Overlay Engine treats it as unavailable rather than authoritative.
 
**Finding 2 —** Overlay Engine's cloud and live branches lack a stated internal isolation requirement
 
Both branches run inside one component; this is the correct place for the reconciliation job to live, but it needs an explicit internal design requirement that a defect or malformed input on one branch cannot affect the other.
 
Action: see hara.md item #1 and Section 9 — an explicit internal freedom-from-interference requirement is needed within the Overlay Engine's own design.
 
**Finding 3 —** Live-to-cloud matching tolerance and multi-candidate assignment are undefined
 
No stated distance/position tolerance, and no stated behavior for multiple simultaneous live candidates against multiple cloud entries.
 
Action: define an explicit matching tolerance and assignment algorithm, considering false-match, false-non-match, and cross-match failure modes.
 
**Finding 4 —** Sentinel value representation not yet specified
 
interfaces.md #6 specifies which fields populate per entry_type, but not the actual sentinel value used, beyond "never a valid-looking number."
 
Action: specify the exact sentinel representation per field type (e.g., NaN, a reserved out-of-range value, or an explicit valid/invalid flag per field) and require every consumer to gate on entry_type before reading any other field.
 
**Finding 5 —** CLOUD_CLEAR determination correctness is not yet verified
 
CLOUD_CLEAR is now the sole basis for a CLEARED report — a defect in reading perception_health_state or confirming position against the cloud-expected location produces a false CLEARED with no other check catching it before it reaches the cloud database.
 
Action: verify the Overlay Engine's CLOUD_CLEAR logic against the actual design; consider whether a single cycle's confirmation is sufficient or whether persistence across multiple cycles/passes should be required before asserting CLOUD_CLEAR.
 
**Finding 6 —** Pothole Cloud-Overlay Engine's own real-time budget is undefined
 
Its matching-logic compute cost scales with cloud-advisory list size and simultaneous live-candidate count, with no stated worst-case bound.
 
Action: define a worst-case complexity bound and real-time budget specific to this engine.
 
**Finding 7 —** Interface #6's shared-memory broadcast has no stated synchronization mechanism
 
One writer, two readers, no versioning or atomic-write guarantee stated — a torn-read risk.
 
Action: specify a synchronization mechanism (double-buffering, atomic swap, or a sequence number readers check before trusting a message) as a functional/software requirement.
 
**Finding 8 —** Healing Engine's multi-report aggregation logic is unspecified, for both adding and clearing entries
 
Threshold, time window, and independence-of-reports are all undefined for how the cloud decides to add a pothole to the database from vehicle reports, and separately, how it decides to clear one. This covers two distinct risks that share the same root cause: correlated false CLEARED reports (multiple vehicles all avoiding the same real pothole, so none confirm it via jerk, discussed under hara.md item #5), and correlated or even single-report false ADDED entries (a ghost detection reported once, or the same visual artifact independently hallucinated by multiple vehicles under similar conditions, persisting in the shared database until enough independent CLOUD_CLEAR confirmations accumulate). hara.md item #2 (ghost pothole) was reclassified in that document's latest revision specifically because this aggregation gap, not Perception's own discrimination accuracy, is this feature's actual exposure to false-positive propagation.
 
The central open question is what "independent" or "correlated" actually means here — matching environmental conditions (lighting, shadow angle, weather), same time of day, spatial clustering of reporting vehicles, or something else. This is currently undefined anywhere in these documents and needs to be resolved explicitly and unambiguously in the functional requirements document — not assumed or left implicit.
 
Action: define the aggregation logic explicitly for both directions (add and clear), including a concrete, testable definition of what makes multiple reports independent enough to count toward a threshold rather than correlated. Likely modeled on existing automotive fault-logging/clearing debounce patterns (confirm-before-log, confirm-before-clear over N occurrences) rather than invented from scratch for this feature.

**Finding 9 —** Verification Engine has no defined behavior for multiple simultaneous in-flight entries
 
If two or more pothole entries are awaiting jerk confirmation concurrently, there's no stated assignment logic to ensure a jerk event is correlated to the correct one — the Verification-Engine-side counterpart to Finding 3.
 
Action: define an explicit assignment approach (e.g., nearest-position-match among pending entries, with a defined tie-breaking rule) rather than leaving "correlate the jerk event" as an implicit single-candidate assumption.