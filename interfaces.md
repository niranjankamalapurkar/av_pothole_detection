This document defines the interfaces this feature owns. See block_diagram.md's "Architectural Basis" section for the cited reasoning behind the overall shape of this design — not repeated here.

------------------------------
## 1. Perception → Pothole Cloud-Overlay Engine

* Protocol: Automotive Ethernet / PCIe
* Payload: array[pothole_id, distance_to_pothole_m, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score]
* Rationale: Basic detection and tracking information only — distance, dimensions, confidence. pothole_confidence_score is a per-detection quality signal, independent of Perception's own system-level compute health — the two are separate signals, not derived from one another. Empty when Perception has dropped pothole classification under compute pressure (its own internal prioritization).

------------------------------
## 2. Jerk Sensor → Verification Engine

* Protocol: CAN-FD
* Payload: timestamp, sensor_health_state, jerk_magnitude, jerk_confidence_score
* Rationale: Physical confirmation, consumed only at validation time. sensor_health_state = FAULTED is treated as unknown, not a confirmed absence of jerk.

------------------------------
## 3. Localization Module → Pothole Cloud-Overlay Engine

* Protocol: Internal IPC / Automotive Ethernet
* Payload: timestamp, latitude, longitude, altitude, hd_map_lane_id, localization_confidence
* Rationale: Used for two purposes: matching a live detection against the cloud-advisory list by position, and positioning a cloud-only entry directly when no usable live detection exists. **localization_confidence lets this engine discount matching/positioning when position accuracy is degraded.**

------------------------------
## 4. Localization Module → Verification Engine

* Protocol: Internal IPC / Automotive Ethernet
* Payload: timestamp, latitude, longitude, altitude, hd_map_lane_id, localization_confidence
* Rationale: Attaches report coordinates for LIVE_ONLY and LIVE_ENRICHED entries (interface #6), which do not carry lat/lon — Path Planner has no use for it there, but the cloud report always needs a location. On degraded confidence or lost position fix, Verification suppresses reporting for that cycle rather than sending an unreliable location.

------------------------------
## 5. Connectivity Manager → Pothole Cloud-Overlay Engine

* Protocol: Internal IPC / Shared Memory
* Payload: array[pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm, recommended_speed_limit]
* Rationale: The cloud-advisory list this engine matches against or falls back to. recommended_speed_limit originates from a more sophisticated, non-real-time cloud-side algorithm (Map Update & Healing Engine), distinct from anything computed at the edge — the edge computes nothing of the kind. Intermittent/partial/corrupted-in-transit cloud data is rejected outright at the Connectivity Manager and never forwarded here.

------------------------------
## 6. Pothole Cloud-Overlay Engine → World Model Builder, Pothole Cloud-Overlay Engine → Verification Engine

* Protocol: Internal IPC / Shared Memory (single broadcast, both consumers read the same message). 
* Payload: entry_type (LIVE_ONLY | LIVE_ENRICHED | CLOUD_SUBSTITUTED), pothole_id, distance_to_pothole_m, lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score, recommended_speed_limit. 

    Note: Fields not applicable to a given entry_type are left at a defined default (e.g., distance_to_pothole_m unset for CLOUD_SUBSTITUTED, since there is no live range measurement; recommended_speed_limit unset for LIVE_ONLY, since no cloud match was found) rather than the message taking a different shape.

* Rationale: A single message definition broadcast once is cheaper on a shared bus — each additional message type carries its own definition and parsing overhead, which costs more in practice than the bytes saved by trimming an occasionally-unused field. Both consumers read the same broadcast: the World Model Builder uses whichever fields are relevant to it and ignores lat/lon when a live distance is present, and Verification uses entry_type plus jerk confirmation (interface #2) the same way regardless of which fields happen to be populated. The entry_type tag alone tells Verification whether the cloud already expected this pothole, without a separate connection to the cloud's expected-list.

------------------------------
## 7. Verification Engine → Connectivity Manager (Telemetry)

* Protocol: Internal IPC / MQTT
* Payload: timestamp, pothole_id, event_type (NEW | ACTIVE | CLEARED), lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score
* Rationale: entry_type (interface #6) plus jerk confirmation (interface #2) determines NEW (LIVE_ONLY, jerk confirms), ACTIVE (LIVE_ENRICHED or CLOUD_SUBSTITUTED, jerk confirms), or CLEARED (LIVE_ENRICHED or CLOUD_SUBSTITUTED, jerk contradicts) — combined with location (interface #4) and offloaded for upstream transmission.

------------------------------
## Baseline Dependencies (not defined by this feature)

**A. Vehicle Speed Calculator → Central Path Planner**
* Payload: vehicle_speed_mph, at minimum
* Rationale: Path Planner's own speed input, combined with a world-model pothole entry's distance, dimensions, and confidence (or severity_index/recommended_speed_limit, for cloud-substituted entries) to compute an actual bounded response.

**B. World Model Builder → Central Path Planner**
* Payload: the unified world model
* Rationale: Path Planner's single input, for this feature and every other perceiver. This feature contributes to the world model (interface #6) but does not define this connection or the builder's own ingestion contract.

**C. Central Path Planner → Vehicle Controls**
* Payload: actuation commands (steering, acceleration)
* Rationale: Pre-existing vehicle architecture, independent of this feature's existence. Previously listed as a feature-defined interface in earlier versions of this document — reclassified here, since this feature never actually had a basis to define it.

---

## Version History

**Version 1 — Initial draft.** Ten interfaces, linear pipeline, no health/confidence fields, no gating, no baseline dependencies documented.
 
**Version 2.** Added health/confidence signaling; an Enablement Gate; an Output Arbitrator; a three-way split of Geotagging & Verification Engine; a coarse braking_level category; explicit rejection of intermittent/partial cloud data.
 
**Version 3.** 
 - Feature updated to be inline with state of the art AV architecture. World Model Builder introduced as explicit baseline architecture. Path Planner's only input became the World Model Builder's output.
 - Introduced Pothole Cloud-Overlay Engine.
 - Central Path Planner → Vehicle Controls moved from a feature-defined interface to Baseline Dependency
