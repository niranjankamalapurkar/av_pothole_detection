This document defines the interfaces this feature owns. See block_diagram.md's "Architectural Basis" section for the cited reasoning behind the overall shape of this design — not repeated here.

------------------------------
## 1. Perception → World Model Builder (Live layer)

* Protocol: Automotive Ethernet / PCIe
* Payload:
    1. timestamp
    2. perception_health_state (NOMINAL | DEGRADED | FAULTED)
    3. array[distance_to_pothole_m, grid_resolution_cm, matrix[N][M] depth_values, pothole_confidence_score]
* Rationale: A local 2.5D depth-grid patch (N×M relative elevation/depth values at grid_resolution_cm spacing), not a bounding-box summary — this is the accurate representation for how a specific tire would actually clip or drop into the anomaly, and it requires no object-level matching downstream, since grid cells fuse spatially rather than needing to be correlated by identity. No persistent identifier: grid-based fusion identifies locations by spatial cell, not object ID, so Perception does not need to track candidate identity across cycles. distance_to_pothole_m remains relative (vehicle-to-patch), since Perception does not perform its own global positioning — the World Model Builder converts this to an absolute patch center (Baseline Dependency E) when populating the Live layer. pothole_confidence_score and perception_health_state remain independent signals. Empty array when Perception has dropped pothole classification under compute pressure. N and M are intentionally small (a local patch, not a wide-area scan) — exact dimensions are a validation-team deliverable, not asserted here; a larger patch is a real per-report bandwidth cost against a 3-field bounding-box summary, accepted deliberately for the accuracy gain.

------------------------------
## 2. Connectivity Manager → World Model Builder (Prior layer)

* Protocol: Internal IPC / Shared Memory
* Payload: timestamp, array[pothole_id, patch_center_lat, patch_center_lon, grid_resolution_cm, matrix[N][M] depth_values, recommended_speed_limit]
* Rationale: The cloud-advisory list, populating the Prior layer directly as a depth-grid patch per entry — no separate matching component sits between this and the World Model Builder's fusion mechanism. pothole_id is retained here specifically because it reflects the cloud database's own internal bookkeeping (needed for the cloud's own update/clear operations), unlike interface #1, which carries no such identifier. patch_center_lat/lon is absolute here, since it originates from the cloud's own database. timestamp indicates list freshness — no staleness threshold defined yet (open item, see fmea.md). recommended_speed_limit originates from a more sophisticated, non-real-time cloud-side algorithm. Authentication/anti-spoofing validation belongs at the Connectivity Manager (Baseline Dependency D), not reimplemented here. Intermittent/partial/corrupted-in-transit cloud data is rejected outright at the Connectivity Manager and never forwarded here.

------------------------------
## 3. World Model Builder → Pothole Observation Reporter

* Protocol: Internal IPC / Shared Memory
* Payload: array[patch_center_lat, patch_center_lon, grid_resolution_cm, matrix[N][M] fused_depth_values, fused_confidence_score], timestamp
* Rationale: The World Model Builder's own fusion of the Live and Prior layers (interfaces #1, #2), per grid cell — combining two depth-grid patches into one fused patch is elementwise matrix fusion (e.g., Bayesian/evidential combination per cell), the same general-purpose mechanism the builder already applies to its other layers, not a bolt-on for this feature. patch_center_lat/lon here is absolute, not the grid's (typically ego-relative) internal representation — the World Model Builder is assumed to perform that local-to-global conversion internally, using its own Localization access (Baseline Dependency E), since it already needs a global reference to correlate against HD maps for its other functions. This is an assumption about baseline behavior this feature cannot verify — if it does not hold, the Reporter would need its own direct Localization access after all. Cloud-side fleet fusion (block_diagram.md Section 5) depends on this: the Healing Engine cannot correlate observations across vehicles without each one carrying an absolute location. This is the only place fusion happens; there is no upstream object-level matching step to define.

------------------------------
## 4. Pothole Observation Reporter → Connectivity Manager (Telemetry)

* Protocol: Internal IPC / MQTT
* Payload: timestamp, patch_center_lat, patch_center_lon, grid_resolution_cm, matrix[N][M] depth_values, fused_confidence_score, vehicle_id
* Rationale: A raw observation — the fused depth-grid patch and its position — not a status assertion; this component does not determine NEW / ACTIVE / CLEARED, that determination is made only after aggregation across vehicles (Map Update & Healing Engine, block_diagram.md Section 5). vehicle_id is included specifically so the cloud can distinguish genuinely independent reports from the same vehicle reporting repeatedly — the load-bearing signal for the aggregation logic's independence requirement (see fmea.md Finding 8, now the single most consequential open item in this design, since no physical-confirmation backstop exists behind it). Whether a threshold/debounce condition triggers this report at all — and its exact parameters — is a validation-team deliverable, not asserted here.

------------------------------
## Baseline Dependencies (not defined by this feature)

**A. Vehicle Speed Calculator → Central Path Planner**
* Payload: vehicle_speed_mph, at minimum
* Rationale: Path Planner's own speed input, combined with a world-model pothole entry to compute an actual bounded response — Path Planner's own inherited logic, not defined by this feature.

**B. World Model Builder → Central Path Planner**
* Payload: the unified world model
* Rationale: Path Planner's single input, for this feature and every other perceiver. This feature contributes two layers to the world model (interfaces #1, #2) but does not define this connection or the builder's own ingestion/fusion contract.

**C. Central Path Planner → Vehicle Controls**
* Payload: actuation commands (steering, acceleration)
* Rationale: Pre-existing vehicle architecture, independent of this feature's existence.

**D. Connectivity Manager's cloud-data authentication/validation function**
* Payload: n/a — an internal security function, not an interface this feature receives data on
* Rationale: The Connectivity Manager is assumed to authenticate and validate all incoming cloud data at the point it receives it from the network, before forwarding to any consumer. Baseline architecture; interface #2 relies on this rather than duplicating validation downstream.

**E. Localization Module → World Model Builder**
* Payload: vehicle position/coordinates, at minimum
* Rationale: The World Model Builder needs this for all layers generally, not specifically for pothole content. New to this list — the prior architecture's dedicated Overlay Engine needed its own direct Localization feed (a feature-defined interface); that need no longer exists once fusion happens inside the World Model Builder's own baseline mechanism.

---

## Version History

**Version 1 — Initial draft.** Perception (forward-only, implicitly dedicated), Jerk/IMU, Sensor Fusion, Localization, Geotagging & Verification Engine, Telemetry & Storage, Cloud Infrastructure, Path Planner, and Vehicle Controls, as a single linear pipeline.

**Version 2.** Added health/confidence signaling; an Enablement Gate; an Output Arbitrator selecting between real-time and cloud-advisory records; a split of the original Geotagging & Verification Engine into three components; a coarse braking_level category; explicit rejection of intermittent/partial cloud data.

**Version 3.** 
 - Feature updated to be inline with state of the art AV architecture. World Model Builder introduced as explicit baseline architecture. Path Planner's only input became the World Model Builder's output.
 - Introduced Pothole Cloud-Overlay Engine.
 - Central Path Planner → Vehicle Controls moved from a feature-defined interface to Baseline Dependency

**Version 4.** 
Current. A fundamental restructuring, following independent review against documented SOTA patterns (multi-layer world-representation fusion; crowdsourced, sensor-independent confidence aggregation):

- Pothole Cloud-Overlay Engine and Verification Engine eliminated as distinct components. Their function is replaced by two layers (Live, Prior) fed directly into the World Model Builder's existing multi-layer fusion mechanism, and a much thinner Pothole Observation Reporter that triggers lightweight reports on significant fused-confidence change — no explicit object-level matching, no entry_type state machine.
- Jerk/IMU sensor removed entirely. Physical confirmation is no longer part of this feature's design. The robustness argument shifts from onboard sensor redundancy to fleet-scale observation diversity — multiple vehicles, multiple times, multiple conditions — consistent with documented crowdsourced road-condition systems [6][7]. This closes the avoidance paradox completely (not just for CLEARED, as Version 4 did, but for NEW/ACTIVE too): a vehicle that visually detects and then successfully avoids a real pothole still reports what it saw, since reporting no longer depends on physical contact at all.
- NEW/ACTIVE/CLEARED determination moved entirely to the cloud's Map Update & Healing Engine, aggregating observation reports across vehicles. A single vehicle no longer asserts a status — it reports an observation. Independence of reports (not just their count) is now the single most safety/quality-relevant open question in this design, since it is the sole defense against correlated false positives with the physical-confirmation backstop removed.
- pothole_id removed from Perception's and the Reporter's outputs — grid-based fusion identifies locations spatially, not by persistent object ID. Retained on cloud-sourced data, where it reflects the cloud database's own internal bookkeeping.
- Localization → World Model Builder reclassified from a feature-defined interface to a baseline dependency — the dedicated component that needed its own direct feed no longer exists.
- Clarification within this version: made explicit that the World Model Builder's fused output to the Pothole Observation Reporter carries absolute lat/lon, not just an internal grid position, and stated as an explicit assumption that the World Model Builder performs the local-to-global conversion itself — cloud-side fleet aggregation depends on this, and it was previously described too vaguely ("already spatially referenced") to make that dependency clear.
- Pothole representation is now a local 2.5D depth-grid patch — a small N×M matrix of relative elevation/depth values at a defined grid resolution, centered on an absolute lat/lon for cloud-sourced or fused entries — consistent with documented 2.5D elevation-map practice for road-surface representation.
- Terminology correction within this version: "costmap" is replaced throughout with "multi-layer/multi-channel world representation." A single flat costmap was never an accurate description — documented multi-layer grid architectures carry distinct content types per layer (occupancy, visibility, ground-surface elevation, friction), not one shared scalar (block_diagram.md reference [11]). This correction does not change any interface's payload or behavior, only the terminology describing the structure it participates in.