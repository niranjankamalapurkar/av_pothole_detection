## 1. Perception System → Enablement Gate

* Protocol: Automotive Ethernet / PCIe
* Payload: timestamp, bounding_box, est_length_cm, est_width_cm, est_depth_cm, visual_confidence_score, perception_health_state, rear_awareness_confidence
* Rationale: Delivers forward-looking visual object detection, bounding geometry, estimated dimensions, optical confidence level, and compute health state from Perception's fused camera/LiDAR pipeline to the Enablement Gate, which decides whether this data proceeds to Sensor Fusion at all. Perception is a single, unified stack covering the full 360-degree field (front/side/rear sensor zones fused into one world model), not separate subsystems per direction — rear_awareness_confidence is part of that same unified output, not a signal from a separate system. perception_health_state (NOMINAL / DEGRADED / FAULTED) signals reduced compute (e.g., thermal throttling, fail-operational conditions); pothole detection is the lower-priority consumer of this shared resource and is gated off first under compute pressure, before Sensor Fusion spends any cycles on it.

------------------------------
## 2. Enablement Gate → Sensor Fusion

* Protocol: Internal IPC / Shared Memory
* Payload: timestamp, bounding_box, est_length_cm, est_width_cm, est_depth_cm, visual_confidence_score
* Rationale: Forwards Perception's detection payload onward only when the gate's state is ENABLED — perception_health_state and rear_awareness_confidence are not forwarded, since Sensor Fusion has no further use for them once the gate has already acted on them. When DISABLED, no message is sent and Sensor Fusion does not run for that cycle — no fusion compute is spent on unreliable input, and neither the real-time avoidance path nor the cloud-reporting path is exercised.

------------------------------
## 3. Jerk Sensor → Sensor Fusion

* Protocol: CAN-FD
* Payload: timestamp, sensor_health_state, jerk_magnitude, jerk_confidence_score
* Rationale: Delivers reactive physical impact readings, vertical spike magnitudes, and IMU operational status. Only consumed by Sensor Fusion when the Enablement Gate is ENABLED (interface #2) — the Jerk sensor's own health does not gate the whole pipeline, since a faulted Jerk sensor still allows visual-only detection to proceed. sensor_health_state = FAULTED is treated by Sensor Fusion as an unknown reading, not as a confirmed zero-jerk value — it must not be allowed to suppress an otherwise-valid visual confidence score. Physically confirms contact with a road irregularity, a function camera/LiDAR cannot perform even in principle, and fails independently of the visual-artifact conditions that affect camera/LiDAR.

------------------------------
## 4. Sensor Fusion → Geotagging & Verification Engine

* Protocol: Internal IPC / Shared Memory
* Payload: timestamp, fused_pothole_id, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score
* Rationale: Passes the unified pothole observation and its calculated pothole_confidence_score (fusing visual and IMU feeds) to be geotagged and verified. Only runs at all when the Enablement Gate (interface #2) is ENABLED.

------------------------------
## 5. Vehicle State / Odometry → Geotagging & Verification Engine

* Protocol: CAN-FD
* Payload: timestamp, vehicle_speed_mph
* Rationale: Standardized broadcast from the vehicle's central state estimator. Crucial for context: a high jerk confidence at 15 mph means a much deeper pothole than a high jerk confidence at 70 mph. Distinct from the Vehicle Speed Calculator's baseline connection directly to the Central Path Planner (see Baseline Dependencies below), which exists independent of this feature.

------------------------------
## 6. Localization Module → Geotagging & Verification Engine

* Protocol: Internal IPC / Automotive Ethernet
* Payload: timestamp, latitude, longitude, altitude, hd_map_lane_id, localization_confidence
* Rationale: Supplies pinpoint spatial positioning data at millisecond precision to accurately anchor detected edge anomalies to geographic coordinates. localization_confidence (analogous to the Jerk sensor's sensor_health_state) allows the Geotagging & Verification Engine to detect drift or a lost position fix, rather than trusting coordinates unconditionally. This does not route through the Enablement Gate — it governs the Geotagging & Verification Engine's own internal split in behavior (see interface #7).

------------------------------
## 7. Geotagging & Verification Engine → Central Path Planner
* Protocol: Internal IPC / Shared Memory
* Payload: timestamp, pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score, encounter_speed_mph
* Rationale: Streams real-time, locally confirmed pothole detections directly to the Path Planner for immediate edge-level trajectory adjustment or avoidance maneuvers. Path Planner has no use for absolute coordinates — it needs to know how far ahead the hazard is, not where it sits on a map — so lat/lon is not part of this payload at all; absolute coordinates remain only on the geotagging/cloud-reporting path (interface #8), which is what actually needs them. distance_to_pothole_m is computed from relative sensor ranging, not GPS, and is exactly the quantity the pothole-avoidance deceleration-bound calculation depends on. recommended_target_speed_mph is a suggested comfortable-traversal speed, computed here rather than recomputed by the time-constrained Path Planner — advisory only; Path Planner is not obligated to achieve it and retains full authority over the actual response, bounded by its own magnitude ceiling and arbitration logic. This path operates independently of the cloud/advisory path (interfaces #9-#11) — neither the absence nor the incorrectness of cloud data ever gates or overrides this path, and this path has no dependency on localization accuracy at all, since it carries no coordinates.

------------------------------
## 8. Geotagging & Verification Engine → Connectivity Manager (Telemetry)
* Protocol: Internal IPC / MQTT
* Payload: timestamp, pothole_id, event_type (NEW | ACTIVE | CLEARED), lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score, encounter_speed_mph
* Rationale: Offloads geotagged pothole reports and verification status updates (ACTIVE or CLEARED) to the telemetry layer for upstream transmission to the cloud. Only sent when localization_confidence (interface #6) indicates a trustworthy position fix — geotagging and cloud reporting are suppressed on degraded/lost localization, distinct from the real-time path (interface #7), which continues regardless.

------------------------------
## 9. Connectivity Manager → Central Path Planner
* Protocol: Cellular (5G/LTE) / gRPC or REST (cached locally)
* Payload: array[pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm, severity_index, recommended_max_speed]
* Rationale: Streams the cloud-maintained localized list of known upcoming potholes directly to the Central Path Planner to enable proactive, forward look-ahead trajectory planning (lane changes, nudges, deceleration). Intermittent, partial, or corrupted-in-transit data over the cellular/Wi-Fi link is rejected outright by the Connectivity Manager and never forwarded here — this array only ever contains complete, validated records.

------------------------------
## 10. Connectivity Manager → Geotagging & Verification Engine
* Protocol: Internal IPC / Shared Memory
* Payload: array[pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm]
* Rationale: Feeds the cloud's list of expected road conditions into the Verification Engine, allowing real-time edge sensors to verify whether an expected pothole remains ACTIVE or has been patched (CLEARED). As with interface #9, intermittent/partial/corrupted-in-transit cloud data is rejected outright at the Connectivity Manager and never forwarded here.

------------------------------
## 11. Central Path Planner → Vehicle Controls
* Protocol: FlexRay / CAN-FD
* Payload: actuation_timestamp, target_steering_angle, target_acceleration
* Rationale: Transmits the final arbitrated steering and braking commands to drive-by-wire actuators.

------------------------------
## Baseline Dependencies (not defined by this feature)

Path Planner consumes additional data from Perception and the Vehicle Speed Calculator beyond what this feature routes through its own pipeline. Both sources are already defined elsewhere in this document; what remains unspecified is only the protocol and full payload of these particular connections, since they are baseline vehicle functionality, not specific to this feature.

**A. Perception System → Central Path Planner**
* Payload: general world-model content, including rear/adjacent-lane traffic state (trailing-vehicle distance, relative speed), at minimum
* Rationale: The Path Planner needs this for any lane-change or braking decision, with or without this feature. Sourced from the same unified Perception system as interface #1, not a separate subsystem.

**B. Vehicle Speed Calculator → Central Path Planner**
* Payload: vehicle_speed_mph, at minimum
* Rationale: The Path Planner needs its own speed input for general driving decisions, independent of this feature. Distinct from interface #5 above (Vehicle Speed Calculator → Geotagging & Verification Engine), which this feature does define, for pothole-severity context specifically.

---

## Version History

**Version 1 — Initial draft.** Ten interfaces, linear pipeline (Perception/Jerk → Sensor Fusion → Geotagging & Verification Engine → Path Planner / Connectivity Manager → Cloud), no health/confidence fields beyond the Jerk sensor's, no gating, no baseline dependencies documented.

**Version 2 — Current.** Following safety analysis of Version 1:
- perception_health_state and rear_awareness_confidence added to the Perception interface (#1); new interfaces added ahead of Sensor Fusion for the Enablement Gate, which forwards Perception's payload only when compute health and rear-awareness confidence permit. Both fields come from the same unified Perception system — an earlier draft of this revision modeled rear-awareness as a separate, undocumented signal source; corrected to reflect that modern AV perception is a single fused 360-degree system, not separate subsystems per direction.
- localization_confidence added to the Localization interface (#6), enabling the Geotagging & Verification Engine to detect drift or loss rather than trust coordinates unconditionally.
- The real-time path to the Path Planner (#7) now states explicitly that lat/lon may be absent without blocking the avoidance decision, and that response magnitude is bounded to a fixed ceiling regardless of arbitration weighting — no priority field needed.
- Connectivity Manager's handling of intermittent/partial/corrupted-in-transit data made explicit on interfaces #9 and #10 (rejected outright, not partially processed).
- A "Baseline Dependencies" section added, documenting two connections (Perception's general world-model output, a direct vehicle-speed feed) this feature relies on but does not define, both terminating at Path Planner.
- An explicit priority-tagging field on the real-time interface was evaluated during this revision and deliberately not adopted, for the same reason noted in the block diagram's version history — the magnitude bound achieves the same goal without adding a new field or new Path Planner logic.
- All interfaces renumbered to accommodate the above; interface numbers are not stable across versions.