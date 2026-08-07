## Ghost Pothole (False-Positive Perception Classification)

### 1. Function & Specification
Perception + Sensor Fusion shall classify a forward road-surface region as a pothole, with an associated confidence score, sufficient for the Path Planner to treat it as actionable — under normal, non-faulty camera/LiDAR operation.

### 2. Related Research / Existing Benchmarks
Two points of reference from outside this repo inform this analysis. Both are cited as background evidence that the general direction is sound — neither is a validated target for this system, since both come from a different sensor setup and a different feature than the one analyzed here.
 
* Published work fusing camera-based detection (YOLOv8) with 3D point-cloud depth data for pothole detection reports precision improving from 89.3% (camera-only) to 95.8% with depth fusion specifically by filtering out road patches, manhole covers, and stains that camera-only detection misclassifies as potholes (Zhong et al., *Scientific Reports*, 2025). This is evidence that requiring depth corroboration before elevating a camera-only detection is a validated direction in the literature, not proof this system will hit the same numbers — different sensor rig, small test road.
* NHTSA's proposed AEB rule for light vehicles specifies staged false-positive test scenarios (a steel trench plate, driving between two parked vehicles — both visually resembling a hazard that isn't one) and requires the vehicle not brake in excess of 0.25g (≈2.45 m/s²) during those tests. This is a real regulatory precedent for capping response severity on a false positive; it is discussed further, and specifically why it is not adopted directly, in Risk Acceptance Criteria below.

### 3. Triggering Conditions
- Harsh shadows from low sun angle
- Tar-seal pavement repairs (visually similar dark patch)
- Wet or reflective asphalt altering perceived depth
- Manhole covers and drain grates
- Debris, leaves, or tire skid marks
- Faded lane paint under glare
- Motion blur at highway speed
- Partially occluded or dirty lens (borderline — degraded, not failed, sensor)
- Heavy rain or snow obscuring surface texture

### 4. Vehicle-Level Hazardous Behavior
The Path Planner receives a request to nudge or brake for a hazard that isn't there. SG-01's arbitration gate still applies, so a false positive cannot cause a swerve into an occupied lane. The genuinely unsafe outcome is narrower than "false detection": it is **UNNECESSARY HARD BRAKING triggering a rear-end collision from a following vehicle.** This is the metric the acceptance criterion targets in Step 5 — not a generic classification false-positive rate.

### 5. Known / Unknown Categorization
* Known Unsafe: The triggering conditions in Step 2. Well-documented, industry-recognized challenges for camera/LiDAR negative-obstacle detection. Known to exist; performance against them must still be validated.
* Unknown Unsafe: Combinations and edge cases not yet enumerated — specific pavement materials, glare geometry from particular structures, sensor-fusion interaction edge cases. Found via exposure (Step 7), not brainstorm.

### 6. Risk Acceptance Criteria
A longer lead distance (earlier, confirmed detection) directly buys a lower required deceleration. Late detection forces hard braking for the same speed change; early detection allows gradual braking, which also gives following traffic more time to react and closes distance more gently, reducing rear-end collision risk. The acceptance criteria below are built around this relationship rather than treating braking rate as a separate, arbitrary number.
 
Speed bands used throughout this criteria set:
* **Highway (~55–75 mph):** highest closing speed, lowest controllability — tightest acceptable targets.
* **Arterial (~35–55 mph):** moderate closing speed and controllability.
* **Urban (<35 mph):** lowest closing speed, highest controllability — shorter following distances offset by lower severity.
Two linked criteria, per speed band:
 
1. False-actionable rate:
Rate of ghost-pothole detections that trigger ANY braking/avoidance response shall not exceed [X] per [Y] vehicle-miles per band (TBD, from benchmark or policy).
2. Maximum deceleration cap for perception-only, uncorroborated response:
Rate of ghost-pothole detections that trigger a braking response above [a_max] shall not exceed [X_hard] — and more importantly, [a_max] itself is bounded below, not derived from an internal comfort preference alone.
**Reference point:** NHTSA's proposed AEB rule for light vehicles specifies staged false-positive test scenarios (a steel trench plate, driving between two parked vehicles — both visually resembling a hazard that isn't one) and requires the vehicle not brake in excess of 0.25g (≈2.45 m/s²) during those tests. This is a real regulatory precedent for capping response severity when the triggering condition turns out not to be a genuine hazard.
 
That number is not directly adopted here, and there is a specific reason it should not be: AEB's 0.25g allowance reflects a severity asymmetry where missing a real forward collision is judged far worse than an unnecessary partial brake, so some false-positive braking is tolerated as the cost of catching real threats. Ghost pothole has the opposite asymmetry. Driving over an undetected pothole is a low-severity outcome — this is exactly why HARA rated the underlying detection chain QM, not ASIL — while an unnecessary hard brake that triggers a rear-end collision can be a *more* severe outcome than the hazard being avoided. Because the false-positive cost here can exceed the false-negative cost (the reverse of AEB's justification), [a_max] should be set at or below 2.45 m/s², treating the AEB figure as an upper ceiling this system should not need to approach rather than a target to match.
 
X, Y, X_hard, and the exact value of [a_max] remain TBD — they need a validation-team-derived benchmark, not an assumed figure. What is fixed by this document is the structure and the direction of the argument: false-actionable rate and the deceleration cap are the two linked criteria, and the cap is bounded above by AEB precedent for reasons specific to this hazard's comparatively lower severity, not adopted from it directly.
 
### 7. Risk-Reduction Measures (design options — not a committed design)
* Raise the confidence threshold required before the Path Planner treats a detection as actionable.
* Require multi-frame temporal confirmation as the vehicle approaches — a single-frame shadow flicker should not be sufficient; confirm across frames at closing range.
* Enforce [a_max] as a hard actuation-level cap for any response triggered by an uncorroborated, perception-only detection — not a soft preference the software can override under urgency. Lead distance then becomes a derived quantity, not a separate rule: given the cap and the lead-distance/deceleration relationship described in Risk Acceptance Criteria above, the achievable speed reduction for a late detection is simply smaller than for an early one. A late detection should yield partial mitigation within the cap, not a breach of the cap and not a refusal to act.
* Known limitation: the Jerk/IMU sensor CANNOT help here. It only confirms a pothole after driving over it, so it is structurally unable to proactively refute a false visual detection ahead of time. Mitigation for this item is confined to the perception/algorithm side (unlike HARA #1, where sensor-fusion cross-checking was available against a malfunction).

### 8. Requirements / Action Items Generated From This Analysis
The two items below are not design options — they are gaps this analysis surfaced in the existing architecture (`block_diagram.md`, `interfaces.md`) and need to be raised as formal action items for the next stage, not treated as one option among several in Section 7 above.
 
* **Cross-modality consistency requirement:** `block_diagram.md`'s Sensor Fusion description treats "Perception" (camera + LiDAR) as a single combined output and never specifies what happens internally when camera and LiDAR disagree. A shadow or tar patch is a purely visual artifact with no true depth signature — LiDAR should already be able to refute it if its "no depth anomaly" signal is weighted correctly inside Perception's internal fusion. 

  * **Action item:** a camera-flagged region shall not be elevated to actionable pothole confidence without corroborating LiDAR depth evidence (an actual negative-depth measurement, not merely the absence of a LiDAR objection).

* **Fusion conflict-resolution policy (currently undefined in interfaces.md):** when camera and LiDAR evidence disagree, the fusion policy must not default to a simple confidence average. 

  * **Action item:** define an explicit resolution rule — options include LiDAR depth-negative evidence vetoing a camera-only detection above a set confidence, or treating disagreement itself as an explicit uncertainty state that forces multi-frame confirmation rather than immediate action. This is a genuine open item in the architecture that should be raised with whoever owns Perception's internal design.

### 9. Verification & Validation Strategy
- Scenario-based test catalog built directly from the Triggering Conditions list above (dedicated shadow-condition, tar-patch, and wet-road test sets).
- Simulation-generated combinations for conditions too rare or costly to stage physically.
- Closed-course testing with staged artifacts.
- Field shadow-mode logging: record what the system would have done without acting on it, for statistical analysis at fleet scale. This is the dedicated post-release monitoring mechanism for this item, referenced in Residual Risk Argument below.
- Incidental note, not a substitute for the above: the cloud Map Update & Healing Engine's existing "no anomaly -> clear" logic is a general feature-level mechanism, not built for this SOTIF item specifically, but it provides a secondary signal that can also help surface ghost-pothole false positives over time.

### 10. Residual Risk Argument
This is not yet an argument that risk has been reduced to an acceptable level — none of the numeric targets exist yet. It is the release gate that must be satisfied before this hazard can be considered adequately mitigated:
 
1. False-actionable rate [X] and deceleration cap [a_max], from Risk Acceptance Criteria, are defined by the validation team and met, with documented statistical confidence, against the test catalog in Verification & Validation Strategy.
2. Every triggering condition listed in Triggering Conditions has documented test coverage — scenario-based, simulation, or closed-course — with pass/fail results recorded.
3. Both action items in Requirements / Action Items Generated (cross-modality consistency requirement, fusion conflict-resolution policy) are implemented in Perception's design and independently verified, not left open at release.

None of these three gates are satisfied by this document — this document defines them; a separate validation/test report closes them.
 
Separately, a dedicated field shadow-mode monitoring (Verification & Validation Strategy) should continue post-release specifically to catch Area 3 (unknown-unsafe) cases that pre-release testing didn't anticipate. This is standard SOTIF practice — Area 3 can never be fully closed before release — not a claim specific to this feature's architecture.

