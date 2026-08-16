## Ghost Pothole (False-Positive Perception Classification)

### 1. Function & Specification
Perception + Sensor Fusion shall classify a forward road-surface region as a pothole, with an associated confidence score, sufficient for the Path Planner to treat it as actionable — under normal, non-faulty camera/LiDAR operation.

### 2. Related Research / Existing Benchmarks
This analysis assumes a mature, multi-sensor perception stack (see Assumptions below). The two reference points here are cited as evidence of the realistic *residual* risk magnitude even at that maturity level — not as justification for adding sensor fusion or temporal confirmation, both of which are assumed already present, not derived by this document.

* Published work fusing camera-based detection (YOLOv8) with 3D point-cloud depth data for pothole detection reports precision of 95.8% with depth fusion (up from 89.3% camera-only), specifically by filtering out road patches, manhole covers, and stains (Zhong et al., *Scientific Reports*, 2025). The relevant takeaway here is not that fusion helps — that is assumed — but that even with fusion, roughly 4% of flagged detections remain false positives. A mature system does not drive this to zero; it drives it to a residual rate that still has to be planned for. This is the evidence that ghost potholes remain a live concern even at Waymo-level maturity, and is why this SOTIF item still exists.
* NHTSA's proposed AEB rule for light vehicles specifies staged false-positive test scenarios (a steel trench plate, driving between two parked vehicles — both visually resembling a hazard that isn't one) and requires the vehicle not brake in excess of 0.25g (≈2.45 m/s²) during those tests. This is a real regulatory precedent for capping response severity on a false positive; it is discussed further, and specifically why it is not adopted directly, in Risk Acceptance Criteria below.

### 3. Assumptions
This analysis is written against a Waymo-maturity perception stack, not a naive single-camera implementation. The following are assumed already true of the baseline system — they are not requirements this SOTIF analysis is deriving:

* The Perception system (Camera + LiDAR, per `block_diagram.md`) already implements properly-weighted cross-modal fusion — a camera-flagged region is not elevated to high confidence without depth corroboration from LiDAR. This is standard practice for a mature AV perception stack (see Related Research above).
* Multi-frame temporal confirmation as the vehicle approaches a candidate detection is already standard practice in the detection pipeline, not a novel measure proposed here.
* `interfaces.md` does not currently document the specific fusion conflict-resolution policy (how camera/LiDAR disagreement is resolved internally). This analysis assumes such a policy exists and is competently designed as part of Perception's internal implementation — it is not specified here, since algorithm design is out of scope for a SOTIF item-level document. If this assumption does not hold — i.e., Perception's internal fusion policy is actually undocumented or unspecified — that is a gap to raise with whoever owns Perception's design, separate from this analysis, and is checked explicitly in Residual Risk Argument below.
* This analysis is scoped to the architecture in `block_diagram.md`, which specifies Camera + LiDAR for Perception, not RADAR. Adding RADAR would be a separate architecture decision; this document does not assume its presence or recommend it (LiDAR is generally the better-suited sensor for shallow, static negative-obstacle depth detection, the specific discrimination task relevant to this hazard).
* The Path Planner has access to real-time trailing-vehicle state (relative distance, relative speed) via its baseline dynamic object tracking function — a general ADS capability, independent of and not built for this pothole-detection feature (consistent with HARA #3's reference to existing dynamic object tracking as a baseline Path Planner input). This analysis assumes that data is available to condition the pothole-avoidance response's magnitude (see Risk Acceptance Criteria and Risk-Reduction / Design Notes below). If it is not actually available in the current implementation, that is a gap to raise separately, and the adaptive requirement below reduces to its most conservative case.

Even given these assumptions hold, mature fusion does not reduce residual risk to zero (Related Research above: ~4% residual false-positive rate even with depth fusion in published work). That residual risk — not the design of the fusion itself — is what this SOTIF item analyzes.

### 4. Triggering Conditions
- Harsh shadows from low sun angle
- Tar-seal pavement repairs (visually similar dark patch)
- Wet or reflective asphalt altering perceived depth
- Manhole covers and drain grates
- Debris, leaves, or tire skid marks
- Faded lane paint under glare
- Motion blur at highway speed
- Partially occluded or dirty lens (borderline — degraded, not failed, sensor)
- Heavy rain or snow obscuring surface texture

### 5. Vehicle-Level Hazardous Behavior
Even under the mature-perception assumptions above, ghost-pothole detections will still occur at some residual rate. When one does, the Path Planner receives a request to nudge or brake for a hazard that isn't there. SG-01's arbitration gate still applies, so a false positive cannot cause a swerve into an occupied lane. The genuinely unsafe outcome is narrower than "false detection": it is **UNNECESSARY HARD BRAKING triggering a rear-end collision from a following vehicle.** This is the metric the acceptance criteria target in Risk Acceptance Criteria below — not a generic classification false-positive rate.

### 6. Known / Unknown Categorization
* Known Unsafe: The triggering conditions in Section 4. Well-documented, industry-recognized challenges for camera/LiDAR negative-obstacle detection. Known to exist even in a mature fusion stack (Related Research above); performance against them must still be validated.
* Unknown Unsafe: Combinations and edge cases not yet enumerated — specific pavement materials, glare geometry from particular structures, sensor-fusion interaction edge cases. Found via exposure (Verification & Validation Strategy below), not brainstorm.

### 7. Risk Acceptance Criteria
A longer lead distance (earlier, confirmed detection) directly buys a lower required deceleration. Late detection forces hard braking for the same speed change; early detection allows gradual braking, which also gives following traffic more time to react and closes distance more gently, reducing rear-end collision risk. The acceptance criteria below are built around this relationship rather than treating braking rate as a separate, arbitrary number.

Speed bands used throughout this criteria set:
* **Highway (~55–75 mph):** highest closing speed, lowest controllability — tightest acceptable targets.
* **Arterial (~35–55 mph):** moderate closing speed and controllability.
* **Urban (<35 mph):** lowest closing speed, highest controllability — shorter following distances offset by lower severity.

Two linked criteria, per speed band:

1. False-actionable rate:
Rate of ghost-pothole detections that trigger ANY braking/avoidance response shall not exceed [X] per [Y] vehicle-miles per band (TBD, from benchmark or policy).
2. Deceleration bound for perception-only, uncorroborated response:
The magnitude of any braking response shall not be a fixed constant — it shall be bounded by the actual rear-collision risk at the moment of response, since that risk, not the pothole itself, is the source of harm identified in Vehicle-Level Hazardous Behavior above:
   - The deceleration magnitude shall never exceed a fixed ceiling [a_ceiling], regardless of context (see Reference point below for how this ceiling is bounded).
   - Below that ceiling, the permitted magnitude shall be a function of trailing-vehicle distance and relative speed (per Assumptions above): tighter — lower magnitude, or no braking at all, accepting the pothole encounter — when a trailing vehicle is close and/or closing fast; may relax toward, but never past, [a_ceiling] when no trailing vehicle is present or it is at a safe distance/speed.
   - When trailing-vehicle state is unavailable (sensor gap, data dropout), the system shall default to the most conservative bound (minimum magnitude, or no braking) — absence of information about rear risk is not evidence of safety.
   - Rate of responses landing in the upper (hard-brake) portion of this bound, near [a_ceiling], shall not exceed [X_hard] — a separate frequency target from #1 above, since even a compliant response can still be uncomfortable or risky if it happens often.

**Reference point:** NHTSA's proposed AEB rule for light vehicles specifies staged false-positive test scenarios (a steel trench plate, driving between two parked vehicles — both visually resembling a hazard that isn't one) and requires the vehicle not brake in excess of 0.25g (≈2.45 m/s²) during those tests. This is a real regulatory precedent for capping response severity when the triggering condition turns out not to be a genuine hazard.

That number is not directly adopted here, and there is a specific reason it should not be: AEB's 0.25g allowance reflects a severity asymmetry where missing a real forward collision is judged far worse than an unnecessary partial brake, so some false-positive braking is tolerated as the cost of catching real threats. Ghost pothole has the opposite asymmetry. Driving over an undetected pothole is a low-severity outcome — this is exactly why HARA rated the underlying detection chain QM, not ASIL — while an unnecessary hard brake that triggers a rear-end collision can be a *more* severe outcome than the hazard being avoided. Because the false-positive cost here can exceed the false-negative cost (the reverse of AEB's justification), [a_ceiling] should be set at or below 2.45 m/s², treating the AEB figure as an upper bound this system should not need to approach rather than a target to match.

**Requirement:** any braking/avoidance response triggered by an uncorroborated ghost-pothole detection shall be bounded by real-time trailing-vehicle risk as described above, and shall never exceed [a_ceiling], itself set at or below the AEB reference ceiling of 2.45 m/s² given this hazard's lower relative severity. X, Y, X_hard, and the exact value of [a_ceiling] remain TBD — they need a validation-team-derived benchmark, not an assumed figure. What is fixed by this document is the structure and the direction of the argument: false-actionable rate and the deceleration bound are the two linked criteria, the bound is context-adaptive to actual rear-collision risk rather than a flat constant, and it is bounded above by AEB precedent for reasons specific to this hazard's comparatively lower severity, not adopted from it directly.

### 8. Risk-Reduction / Design Notes
**Requirement — response priority:** the pothole-avoidance braking/nudge response shall be tagged as Advisory/Comfort priority within the Path Planner's arbitration scheme, strictly below Safety-Critical priority candidates (collision avoidance, pedestrian avoidance, lane-keeping under an active hazard). The Path Planner shall override, defer, or discard the pothole-avoidance response whenever it conflicts with a higher-priority safety-critical candidate. This follows directly from the severity argument in Risk Acceptance Criteria above: driving over an undetected pothole is a low-severity outcome, so there is no safety justification for ever letting this response compete with, delay, or degrade a genuinely safety-critical maneuver. Skipping or overriding it carries no meaningful harm to the passenger.

*Scope note:* this document specifies where the pothole-avoidance response sits in the arbitration hierarchy — lowest tier, always overridable. It does not define the full priority/rating scheme across every type of candidate maneuver the Path Planner arbitrates (collision avoidance vs. lane-keeping vs. comfort features generally). That scheme is Path Planner architecture, spans beyond this single SOTIF item, and belongs in the Functional Safety Concept.

**Requirement — response magnitude:** the deceleration magnitude for any pothole-avoidance response, whenever the priority rule above permits it to execute at all, shall be bounded by real-time trailing-vehicle risk, not a fixed constant — tightening (toward zero, accepting the pothole encounter) as trailing-vehicle distance/relative-speed risk increases, and never exceeding [a_ceiling] regardless of context (see Risk Acceptance Criteria above for the ceiling and its rationale). When trailing-vehicle state is unavailable, the system shall default to the most conservative bound. This directly reflects that rear-collision risk, not the pothole encounter itself, is the actual hazard driving this requirement (Vehicle-Level Hazardous Behavior above). Lead distance remains relevant within this bound as a derived quantity, not a separate rule: given whatever magnitude the trailing-vehicle-risk bound permits, the achievable speed reduction for a late detection is simply smaller than for an early one. A late detection yields partial mitigation within the bound, not a breach of it and not a refusal to act.

* Confidence-threshold and detection-timing calibration are Perception/Fusion design outputs that must be validated to meet the false-actionable-rate and hard-brake-rate targets above — not prescribed here, since specific threshold values are a design/validation deliverable, not a SOTIF-level requirement.
* Known limitation: the Jerk/IMU sensor CANNOT help here. It only confirms a pothole after driving over it, so it is structurally unable to proactively refute a false visual detection ahead of time. Mitigation for this item is confined to the perception/algorithm side (unlike HARA #1, where sensor-fusion cross-checking was available against a malfunction).

### 9. Verification & Validation Strategy
- Scenario-based test catalog built directly from the Triggering Conditions list above (dedicated shadow-condition, tar-patch, and wet-road test sets).
- Simulation-generated combinations for conditions too rare or costly to stage physically.
- Closed-course testing with staged artifacts.
- Field shadow-mode logging: record what the system would have done without acting on it, for statistical analysis at fleet scale. This is the dedicated post-release monitoring mechanism for this item, referenced in Residual Risk Argument below.
- Incidental note, not a substitute for the above: the cloud Map Update & Healing Engine's existing "no anomaly -> clear" logic is a general feature-level mechanism, not built for this SOTIF item specifically, but it provides a secondary signal that can also help surface ghost-pothole false positives over time.

### 10. Residual Risk Argument
This is not yet an argument that risk has been reduced to an acceptable level — none of the numeric targets exist yet. It is the release gate that must be satisfied before this hazard can be considered adequately mitigated:

1. False-actionable rate [X], the deceleration ceiling [a_ceiling], and the trailing-vehicle-risk-adaptive bounding behavior, from Risk Acceptance Criteria, are defined by the validation team and met, with documented statistical confidence, against the test catalog in Verification & Validation Strategy. This includes verifying the pothole-avoidance response is correctly tagged as Advisory/Comfort priority, is actually overridden by the Path Planner in scenarios where it conflicts with a safety-critical candidate, and that its magnitude genuinely tightens with trailing-vehicle risk rather than defaulting to [a_ceiling] regardless of context (Risk-Reduction / Design Notes above).
2. Every triggering condition listed in Triggering Conditions has documented test coverage — scenario-based, simulation, or closed-course — with pass/fail results recorded.
3. The Assumptions in Section 3 — particularly that Perception's internal fusion policy properly weights LiDAR depth corroboration and resolves camera/LiDAR disagreement — are confirmed against the actual design, not merely assumed. If verification shows these assumptions do not hold, this analysis must be revisited, since the acceptance criteria above were derived assuming a mature fusion baseline.

None of these three gates are satisfied by this document — this document defines them; a separate validation/test report closes them.

Separately, a dedicated field shadow-mode monitoring (Verification & Validation Strategy) should continue post-release specifically to catch Area 3 (unknown-unsafe) cases that pre-release testing didn't anticipate. This is standard SOTIF practice — Area 3 can never be fully closed before release — not a claim specific to this feature's architecture.