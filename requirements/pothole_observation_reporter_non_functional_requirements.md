# Pothole Observation Reporter — Non-Functional Requirements

## Scope

Timing and rate requirements for the Pothole Observation Reporter, split out from pothole_observation_reporter_functional_requirements.md following the same functional/non-functional distinction already applied to World Model Builder's requirements: behavioral choices (e.g., "reporting is event-driven, not periodic," POR-REQ-05) stay in the functional document; the specific rate, window, or timing budget behind that behavior belongs here. All requirements below are **QM**, consistent with the Reporter's classification (hara.md Classification Summary).

---

## A. Report frequency and deduplication

**POR-NFR-01 [QM]** — Report deduplication window
The Reporter shall not send more than one report for the same physical candidate location while it remains within the vehicle's perception coverage. A position last reported shall be retained in a suppression record for a duration derived dynamically as (perception coverage radius) ÷ (current vehicle speed), after which it expires and a fresh observation at that position may be reported again.

*Rationale:* per world_model_builder_non_functional_requirements.md WMB-NFR-05's design principle (this feature's components use position, not object identity, wherever possible), deduplication is done by checking whether a position has already been reported recently — not by tracking a candidate's identity across cycles. The suppression window is not a fixed number; it is derived from how long a given position can plausibly remain within view, which depends on both the perception system's spatial coverage and the vehicle's current speed.

*Basis:* documented BEV-based perception systems (the architectural family this design's Live layer draws from, block_diagram.md Architectural Basis) typically cover a 50–100 m radius around the vehicle, with specialized long-range variants extending further for highway/trucking use cases not applicable to this feature's ODD [1]. Using this range: at 15 m/s (~34 mph, typical urban/suburban driving), a 50–100 m coverage radius gives a suppression window of roughly 3.3–6.7 seconds; at 30 m/s (~67 mph, highway), roughly 1.7–3.3 seconds. This replaces sending a report every fusion cycle (10–30 Hz, per world_model_builder_non_functional_requirements.md WMB-NFR-04) with at most one report per multi-second window per position — a substantial, physically-derived reduction, not an arbitrary rate cap.

*Testable as:* hold a candidate stationary in the fused output at a fixed position while varying vehicle speed; verify the suppression window shrinks and expands in inverse proportion to speed, and matches the formula's predicted value at several sample speeds (e.g., 5 m/s, 15 m/s, 30 m/s) within an acceptable margin for grid-resolution/timing-tick granularity.

*Traces to:* pothole_observation_reporter_functional_requirements.md POR-REQ-05 (the behavioral choice this requirement provides the timing budget for); world_model_builder_non_functional_requirements.md WMB-NFR-04, WMB-NFR-05.

---

## References

[1] Documented BEV-based perception typically covers ranges below 50 m to 50–100 m for standard on-road planning, with specialized long-range approaches extending to 250 m for highway/trucking-specific needs not applicable to this feature's ODD. https://light.princeton.edu/publication/lrs4fusion/

---

## Open items surfaced while drafting

- **Resolved** — this requirement needs the Reporter to know current vehicle speed; interfaces.md #4 (Vehicle Speed Calculator → Pothole Observation Reporter) now defines this, including a health/validity signal (vehicle_speed_health_state) this requirement's formula depends on being trustworthy. See pothole_observation_reporter_functional_requirements.md POR-REQ-09 for the corresponding suppress-on-invalid-speed behavior, and fmea.md Finding 8 for the residual, upstream question of whether the Vehicle Speed Calculator's actual baseline output includes a compatible signal.