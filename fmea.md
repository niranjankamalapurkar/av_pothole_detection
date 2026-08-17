## Scope

This is an independent, bottom-up Failure Mode and Effects Analysis (FMEA) of the Pothole Detection & Fleet Map Advisory feature, built directly from `block_diagram.md` and `interfaces.md` — not derived from `hara.md` or the SOTIF documents. Its purpose is to stress-test those top-down analyses: HARA and SOTIF were built by brainstorming candidate hazardous effects, which is prone to missing things a brainstorm doesn't think to ask. FMEA instead walks every block in the architecture and systematically asks what happens when it fails, then checks whether that failure mode is already covered.

---

## Method

Every failure mode below is recorded as:

Failure mode:   what fails, in one line.
Category:       Loss of function / Erroneous output / Delayed output / Intermittent operation.
Local effect:   what happens to that block's own output.
System effect:  what happens downstream (Path Planner, cloud, fleet).
Coverage:       COVERED — [document, section] if an existing document already addresses this, or NOT COVERED — see Finding N.

---

## Perception System (Camera + LiDAR)

**1. Failure mode: Complete loss of function (camera/LiDAR fail entirely)**
    Category:      Loss of function
    Local effect:  No pothole-detection output from Perception at all.
    System effect: Depends on fail-safe design — see Finding 1.
    Coverage:      NOT COVERED — see Finding 1

**2. Failure mode: Visual misclassification (shadow, tar patch, wet asphalt)**
    Category:      Erroneous output
    Coverage:      COVERED — hara.md item #2 / ghost_pothole_sotif.md

**3. Failure mode: Miss on a real pothole**
    Category:      Erroneous output (false negative)
    Coverage:      COVERED — hara.md item #4

**4. Failure mode: Degraded output quality in adverse weather**
    Category:      Intermittent operation
    Coverage:      COVERED — folded into triggering conditions for #2/#4 in ghost_pothole_sotif.md

**5. Failure mode: Compute-resource contention with primary object detection**
    Category:      Delayed output / partial loss of function (under load)
    Local effect:  Pothole-classification workload competes with primary object-detection workload if compute is shared.
    System effect: Whether Perception is dedicated or shared compute is undocumented in block_diagram.md/interfaces.md. If shared, this is a new resource-contention hazard, structurally similar to hara.md item #3 (bus bandwidth) but for compute.
    Coverage:      NOT COVERED — see Finding 2

**6. Failure mode: Reduced compute capacity — thermal throttling or other fail-operational degraded-clock-speed condition**
    Category:      Delayed output (primarily) — degrades toward Loss of function if severe enough to miss real-time deadlines entirely.
    Local effect:  Perception's processing latency increases; detection results arrive later relative to vehicle position than under nominal compute conditions. Distinct from Finding 2 above: nothing is competing for the resource, the resource itself has shrunk (e.g., thermal throttling, fail-operational reduced clock speed).
    System effect: Increased processing latency erodes the lead distance the deceleration-bound framework in ghost_pothole_sotif.md Section 7 assumes is available. A detection that would have landed early enough for a comfort-level response under nominal compute could be pushed toward the hard-brake ceiling, or arrive too late to be actionable at all — without the detection itself being wrong, only its timing. No document currently treats compute health as an input to that calculation.
    Coverage:      NOT COVERED — see Finding 10

---

## Jerk/IMU Sensor

**1. Failure mode: Complete loss of function (sensor fails/disconnects)**
    Category:      Loss of function
    Local effect:  No jerk confirmation available to Sensor Fusion.
    System effect: interfaces.md interface #2 includes sensor_health_state, so the signal exists, but no document specifies whether Sensor Fusion actually treats FAULTED as "unknown" rather than "zero jerk" — misreading the two would incorrectly suppress confidence on a real detection.
    Coverage:      NOT COVERED — see Finding 3

**2. Failure mode: Legitimate non-pothole jerk event (speed bump, railroad crossing, expansion joint) misclassified as pothole confirmation**
    Category:      Erroneous output
    Local effect:  Sensor Fusion assigns pothole confidence to a non-hazard.
    System effect: Cannot cause a proactive braking event (jerk only fires on contact), but can cause a false CONFIRMED report to the cloud database, which downstream vehicles then receive as a proactive advisory for a location with no real hazard.
    Coverage:      NOT COVERED — see Finding 4

---

## Sensor Fusion Module

**Failure mode: Software defect in confidence-score calculation itself (not a performance limitation — a genuine logic bug)**
    Category:      Erroneous output (malfunction, HARA category)
    Local effect:  Fusion emits an incorrect confidence score even when both inputs are individually correct.
    System effect: Reaches the Path Planner via the real-time reactive path (interface #6), same path as hara.md item #1; likely disposition is inherited into SG-01 by analogy — but hara.md's Exhaustive Effects List never explicitly enumerates a local Sensor Fusion logic defect, only a corrupted cloud payload (#1) and a corrupted cloud verification-logic defect (#5).
    Coverage:      NOT COVERED as its own hazardous event — see Finding 5

---

## Localization Module

**1. Failure mode: Drift or complete loss of position fix is currently undetectable**
    Category:      Erroneous output (drift) / Loss of function (total loss)
    Local effect:  interfaces.md interface #5 (Localization -> Geotagging & Verification Engine) carries only timestamp, latitude, longitude, altitude, hd_map_lane_id — no health/confidence/accuracy field, unlike the Jerk sensor's sensor_health_state (interface #2). Geotagging & Verification Engine has no way to distinguish good localization from drifted or absent localization from the payload alone.
    System effect: Drift isn't just unhandled downstream — it's undetectable at the interface level. hara.md item #7 (SOTIF) addresses drift's *consequence* (wrong lane association) but not whether the system can detect the condition at all in order to respond to it.
    Coverage:      NOT COVERED — see Finding 6

**2. Failure mode: Real-time reactive path is architecturally coupled to geotagging when it shouldn't need to be**
    Category:      Design/architecture gap (not a component failure — a specification issue)
    Local effect:  Interface #6 (Geotagging & Verification Engine -> Path Planner, the real-time reactive path) bundles lat/lon into the same payload that drives the avoidance decision.
    System effect: The real-time avoidance decision only needs relative information (hazard N meters ahead, in-lane or not) — it does not need absolute coordinates; only geotagging/cloud-reporting does. As currently specified, localization drift or loss would incorrectly affect real-time avoidance capability, when the correct behavior is to suppress geotagging/logging only, while the real-time signal continues on relative sensor data alone. This is true for both drift and total loss — the same disposition applies to both.
    Coverage:      NOT COVERED — see Finding 6

---

## Geotagging & Verification Engine

**Failure mode: Fails to correctly append the Advisory/Comfort priority tag to its own request**
    Category:      Erroneous output (malfunction, HARA category)
    Local effect:  Interface #6's payload (timestamp, pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score, encounter_speed_mph) currently has no priority/severity field at all. The priority tag ghost_pothole_sotif.md Section 8 requires must be appended by the source of the request (this block), not inferred or enforced independently by the Path Planner.
    System effect:  If this block omits the tag or appends it incorrectly, a pothole-avoidance request could reach Path Planner without the priority information arbitration depends on.
    Coverage:      NOT COVERED — see Finding 7

---

## Connectivity Manager / Local Datalogger / Cloud (Database, Healing Engine, Downstream API)

**1. Failure mode: Connectivity Manager's network-down fallback itself fails; Healing Engine threshold defects; database corruption; full cloud outage**
    Category:      Loss of function / Erroneous output (various, this group)
    System effect: All sit entirely within the advisory (QM) layer hara.md item #5 already established cannot elevate risk above baseline — the real-time reactive path never depends on this layer being correct or present.
    Coverage:      COVERED — hara.md item #5 (freedom-from-interference argument generalizes across the whole advisory layer, not just the one failure mode #5 originally described)

**2. Failure mode: Intermittent/flapping cloud connectivity — distinct from a clean up/down transition**
    Category:      Intermittent operation
    Local effect:  Partial, incomplete, or corrupted-in-transit messages are possible without triggering the clean "network down" fallback path, since interfaces.md defines no threshold for when "intermittent" becomes "down."
    System effect: If partial/incomplete data is used rather than discarded, this could feed corrupted or incomplete records into the advisory layer. Still bounded by the same freedom-from-interference argument as #1 above (advisory layer only) — but only if the system actually rejects incomplete data rather than attempting best-effort use of it, which is not currently stated anywhere.
    Coverage:      NOT COVERED — see Finding 9

---

## Central Path Planner

**Failure mode: Correctly receives an Advisory-tagged pothole request but misinterprets or mis-weighs the priority field during arbitration**
    Category:      Erroneous output (malfunction, HARA category)
    Local effect:  Path Planner's priority ordering is violated even though the input was correctly tagged.
    System effect: Directly invalidates the priority requirement ghost_pothole_sotif.md Section 8 depends on. This is distinct from the Geotagging & Verification Engine failure mode above — that one is a bad tag at the source; this one is Path Planner mishandling a good tag.
    Coverage:      NOT COVERED as its own hazardous event — see Finding 7

---

## Vehicle Speed Calculator

**Failure mode: Incorrect ego speed value reported via interface #4**
    Category:      Erroneous output
    Local effect:  Geotagging & Verification Engine receives a wrong vehicle_speed_mph.
    System effect: block_diagram.md notes this value distinguishes pothole severity by speed. It also silently feeds ghost_pothole_sotif.md's speed-band classification and trailing-vehicle-risk deceleration bound — a wrong ego speed could select the wrong band or miscompute the permitted deceleration.
    Coverage:      Malfunction itself is pre-existing baseline vehicle architecture (unaffected by this feature, per hara.md Section 2) — but the dependency is new. See Finding 8.

---

## Vehicle Controls (Actuation)

**Failure mode: Actuator fails to execute a requested deceleration**
    Category:      Loss of function (mechanical/actuation fault)
    Coverage:      Out of scope — pre-existing baseline vehicle HARA territory (brake system integrity), unaffected by this feature, per hara.md Section 2 ("existing item — unaffected").

---

## Findings Summary

**Architecture / Interface Gap (not a component failure mode)**

**Gap: Path Planner's connection to rear and adjacent-lane traffic perception is entirely absent from block_diagram.md and interfaces.md**

This is not specific to the pothole-detection feature — any lane-change or braking decision the Path Planner makes requires awareness of rear and adjacent-lane traffic, so this connection must already exist in the vehicle's baseline architecture. It simply isn't shown in the documents this repo works from, which only depict the pothole-detection-specific perception path.

Detailed failure-mode analysis of that rear/adjacent-lane perception function itself — false positives, false negatives, sensor coverage — is out of scope here; it belongs to the baseline Path Planner's own HARA and FMEA. What is in scope for this feature is the asymmetric residual risk it inherits: a false positive (a phantom rear vehicle) is benign, but a false negative (missing a real, closing rear vehicle) is dangerous, because it compounds directly with the harm mechanism this feature's severity argument is built around (rear-end collision).

Disposition: this feature-level policy is defined in ghost_pothole_sotif.md, not here — see its Assumptions and the third requirement in Section 8 (suppression under degraded rear-awareness confidence). This entry exists to flag the missing connection in the source diagrams, not to re-derive the perception analysis that connection would require.

**Finding 1 —** Perception loss-of-function: fail-safe, not fusion logic

On complete loss of Perception, the system shall not attempt real-time, pothole-specific detection at all — it shall cleanly disable the real-time reactive pothole path rather than have Sensor Fusion try to interpret absent data. Cloud/advisory data may still be used, since it does not depend on this vehicle's own Perception.

Action: add as an explicit fail-safe requirement. No Sensor Fusion ambiguity-handling logic is needed — the simpler fix is to disable the function cleanly, not make Fusion smarter about absence of data.

**Finding 2 —** Perception compute-resource contention

Whether Perception is dedicated or shared compute with primary object detection is undocumented. If shared, this is a new resource-contention hazard, structurally similar to hara.md item #3 but for compute.

Action: clarify in block_diagram.md/interfaces.md. If shared, add a hazardous event to hara.md analogous to item #3, and require pothole-classification workload be scheduled at lower priority than primary object detection — the same Advisory/Comfort vs. Safety-Critical principle established for Path Planner arbitration, applied here to compute scheduling rather than trajectory arbitration.

**Finding 3 —** Jerk sensor_health_state handling unspecified

interfaces.md defines the field but no document specifies that Sensor Fusion must treat FAULTED as "unknown," not "zero jerk."

Action: add as an explicit Sensor Fusion requirement.

**Finding 4 —** Jerk-sourced false positive, a new Ghost Pothole triggering condition

Speed bumps, railroad crossings, and expansion joints can cause a legitimate jerk event misclassified as pothole confirmation, populating the cloud database with a false entry via a root cause the Ghost Pothole SOTIF's (visual-only) triggering-condition list never considered.

Action: add as a triggering condition to ghost_pothole_sotif.md, with scenario coverage in Section 9 — this is a reporting-path effect (false database entry), not a proactive real-time-braking effect, since jerk cannot fire before contact.

**Finding 5 —** Sensor Fusion confidence-logic defect not enumerated

hara.md's Exhaustive Effects List covers a corrupted cloud payload (#1) and a corrupted cloud verification-logic defect (#5), but never a local, vehicle-side Sensor Fusion software defect reaching the Path Planner via the real-time path.

Action: add as an explicit hazardous event to hara.md Section 5; likely disposition is inherited into SG-01 by the same argument as #1, but it should be listed, not assumed by analogy.

**Finding 6 —** Localization drift/loss is undetectable, and the real-time path is wrongly coupled to it

Two compounding issues: (a) interface #5 has no health/confidence field, so Geotagging & Verification Engine cannot detect drift or total loss from the payload alone; (b) even if detected, interface #6 bundles lat/lon into the same real-time payload that drives Path Planner's avoidance decision, when the avoidance decision only needs relative information (hazard ahead, in-lane or not) — absolute coordinates are only needed for geotagging.

Action: (1) add a confidence/health field to interface #5, analogous to the Jerk sensor's sensor_health_state; (2) decouple interface #6 so the real-time avoidance signal does not require valid lat/lon to be actionable; (3) on detected drift or total loss (same disposition for both), suppress geotagging/logging only — the real-time reactive path continues on relative sensor fusion output alone.

**Finding 7 —** Priority tag: source-side and arbitration-side failures are distinct

The priority tag ghost_pothole_sotif.md Section 8 requires must be appended by Geotagging & Verification Engine (the source of the request), not applied or inferred by Path Planner. Interface #6's payload currently has no priority/severity field at all. Two genuine, distinct failure modes exist: the source failing to append the tag correctly, and Path Planner correctly receiving a tag but misinterpreting or mis-weighing it during arbitration.

Action: (1) add a priority/severity field to interface #6's payload, populated by Geotagging & Verification Engine; (2) add both failure modes as explicit hazardous events in hara.md Section 5 — a tagging defect at the source, and an arbitration-weighting defect at the Path Planner — rather than one vague "arbitration fails to apply the tag" entry, since the two have different owners and different root causes.

**Finding 8 —** Undocumented dependency on Vehicle Speed Calculator accuracy

ghost_pothole_sotif.md's speed-band classification and deceleration bound silently assume ego speed is correct; this dependency is not stated anywhere.

Action: state the dependency explicitly in ghost_pothole_sotif.md, or add a cross-reference note. The malfunction itself is baseline vehicle HARA territory, not new to this feature — the reliance on it is what's new.

**Finding 9 —** Intermittent cloud connectivity is a distinct failure mode from clean network-down

interfaces.md's "Network Down -> Local Datalogger" fallback addresses a clean disconnection, not flapping/partial connectivity, where incomplete or corrupted-in-transit messages are more likely and no threshold is defined for when intermittent becomes down.

Action: add an explicit requirement that intermittent/partial cloud data (incomplete messages, integrity-check failures short of full connectivity loss) is rejected/discarded outright, not partially processed. This remains bounded by the same freedom-from-interference argument as hara.md item #5 (advisory layer only) — but only if this rejection behavior is actually verified, not assumed.

**Finding 10 —** Perception compute degradation (thermal throttling, fail-operational reduced clock speed) is not addressed anywhere

Neither block_diagram.md nor interfaces.md defines a health/thermal status signal for the Perception compute platform, unlike the Jerk sensor's sensor_health_state (interface #2). A sustained reduction in available compute doesn't just slow detection down in the abstract — it erodes the lead-distance budget ghost_pothole_sotif.md's deceleration-bound framework (Section 7) depends on, silently pushing responses toward the hard-brake ceiling or past the point of being actionable, without tripping any existing hazard or SOTIF trigger.

Action: (1) define a compute-health/thermal-status signal for the Perception platform, analogous to sensor_health_state; (2) on detecting degraded compute below a defined threshold, disable real-time pothole-specific detection entirely rather than continue operating at silently reduced fidelity — the same disposition already established in Finding 1 for complete Perception loss, applied here to partial/degraded compute rather than total failure; (3) fall back to cloud/advisory-only data, consistent with Finding 1's reasoning that cloud data doesn't depend on this vehicle's own Perception. Related to Finding 2 (compute-resource contention) — both are compute-availability questions, one from sharing, one from degradation, and may warrant a combined compute-health requirement rather than two separate ones.