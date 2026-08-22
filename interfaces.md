This document defines the interfaces this feature owns. See block_diagram.md's "Architectural Basis" section for the cited reasoning behind the overall shape of this design — not repeated here.

------------------------------
## 1. Perception → Pothole Cloud-Overlay Engine

* Protocol: Automotive Ethernet / PCIe
* Payload:
    1. timestamp
    2. perception_health_state (NOMINAL | DEGRADED | FAULTED)
    3. array[pothole_id, distance_to_pothole_m, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score]
* Rationale: Basic detection and tracking information only — distance, dimensions, confidence. pothole_confidence_score is a per-detection quality signal, independent of Perception's own system-level compute health. Empty array when Perception has dropped pothole classification under compute pressure. timestamp enables latency measurement and supports the Overlay Engine's downstream matching logic.

------------------------------
## 2. Jerk Sensor → Verification Engine

* Protocol: CAN-FD
* Payload: timestamp, sensor_health_state, jerk_magnitude, jerk_confidence_score
* Rationale: Physical confirmation, consumed only at validation time. sensor_health_state = FAULTED is treated as unknown, not a confirmed absence of jerk. Verification Engine treats this as an ongoing stream of readings to correlate against, not a single point-in-time value — see interface #7's rationale for how correlation works.

------------------------------
## 3. Localization Module → Pothole Cloud-Overlay Engine

* Protocol: Internal IPC / Automotive Ethernet
* Payload: timestamp, latitude, longitude, altitude, hd_map_lane_id, localization_confidence
* Rationale: Used for three purposes: matching a live detection against the cloud-advisory list by position; positioning a cloud-only entry directly when Perception is degraded (CLOUD_SUBSTITUTED); and confirming the vehicle's position corresponds to a cloud-expected location when determining CLOUD_CLEAR. **localization_confidence lets Pothole Cloud-Overla engine discount matching/positioning when position accuracy is degraded.**

------------------------------
## 4. Localization Module → Verification Engine

* Protocol: Internal IPC / Automotive Ethernet
* Payload: timestamp, latitude, longitude, altitude, hd_map_lane_id, localization_confidence
* Rationale: Live, ongoing position awareness — distinct from the static lat/lon the Overlay Engine includes in an entry (interface #7), which is computed once at detection/substitution time and says where a pothole *should* be. This interface tells Verification Engine where the vehicle *actually is* as jerk events arrive, so it can check the vehicle's current position against an entry's stored lat/lon at the moment of a candidate jerk match. For CLOUD_SUBSTITUTED and CLOUD_CLEAR entries, which carry no live distance measurement, this position check is the only correlation mechanism available at all — the kinematic approach (interface #5) only works for LIVE_ONLY and LIVE_ENRICHED, where a live distance exists.


------------------------------
## 5. Vehicle Speed Calculator → Verification Engine
 
* Protocol: CAN-FD
* Payload: timestamp, vehicle_speed_mph
* Rationale: Two purposes. First, calibrates jerk severity — the same jerk magnitude indicates a much deeper pothole at 15 mph than at 70 mph. Second, combined with an entry's distance_to_pothole_m and timestamp (interface #7), lets Verification Engine estimate roughly when to expect jerk confirmation for a LIVE_ONLY or LIVE_ENRICHED entry, narrowing which jerk event in the stream (interface #2) is the candidate match. This estimate is approximate, not exact — vehicle speed can change between detection and arrival, especially if Path Planner brakes in response to the same detection — which is why it is used alongside, not instead of, the position check (interface #4). This is a feature-defined consumption of a baseline signal; the Vehicle Speed Calculator itself remains baseline vehicle architecture (Baseline Dependency A, feeding Path Planner separately).

------------------------------
## 6. Connectivity Manager → Pothole Cloud-Overlay Engine

* Protocol: Internal IPC / Shared Memory
* Payload: timestamp, array[pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm, recommended_speed_limit]
* Rationale: The cloud-advisory list this engine matches against or falls back to. timestamp indicates how fresh this list is as of receipt — no staleness threshold defined yet (open item, see fmea.md). recommended_speed_limit originates from a more sophisticated, non-real-time cloud-side algorithm (Map Update & Healing Engine), distinct from anything computed at the edge — the edge computes nothing of the kind. The Connectivity Manager is the vehicle's actual network-facing boundary for cloud data — authentication, anti-spoofing, and integrity validation belong there, applied once to all incoming cloud data before it reaches any consumer, not reimplemented separately by every downstream feature (see Baseline Dependency D). This engine relies on that upstream validation rather than duplicating it. Intermittent/partial/corrupted-in-transit cloud data is rejected outright at the Connectivity Manager and never forwarded here. List size directly affects the Overlay Engine's own matching-logic compute cost — see hara.md item #6.

------------------------------
## 7. Pothole Cloud-Overlay Engine → World Model Builder, Pothole Cloud-Overlay Engine → Verification Engine
 
* Protocol: Internal IPC / Shared Memory (single broadcast, both consumers read the same message).
* Payload: timestamp, entry_type (LIVE_ONLY | LIVE_ENRICHED | CLOUD_SUBSTITUTED | CLOUD_CLEAR), pothole_id, lat, lon, distance_to_pothole_m, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score, recommended_speed_limit.
    Field population — lat/lon is now independent of entry_type, populated whenever localization_confidence (interface #3) is sufficient regardless of type, at sentinel only when it is not. All other fields remain entry_type-dependent (exhaustive, not illustrative):
    - LIVE_ONLY: distance_to_pothole_m, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score populated. recommended_speed_limit at sentinel (no cloud match).
    - LIVE_ENRICHED: all of the above populated, including recommended_speed_limit from the matched cloud entry.
    - CLOUD_SUBSTITUTED: est_length_cm, est_width_cm, est_depth_cm, recommended_speed_limit populated from the cloud entry. distance_to_pothole_m, pothole_confidence_score at sentinel (no live measurement exists).
    - CLOUD_CLEAR: recommended_speed_limit and dimensions from the cloud entry being cleared may be populated for reference. distance_to_pothole_m, pothole_confidence_score at sentinel.
    Sentinel values must be unambiguous and never a valid-looking number (e.g., not 0) — exact sentinel representation is an open specification item, see hara.md item #9.
* Rationale: A single message definition broadcast once is cheaper on a shared bus than separate message types per entry_type. Both consumers read the same broadcast: the World Model Builder uses whichever fields are relevant to it and treats CLOUD_CLEAR as a no-op; Verification Engine uses entry_type together with jerk confirmation (interface #2) as two independent signals for status determination (block_diagram.md Section 4), and uses lat/lon together with live Localization (interface #4) and, where a live distance exists, vehicle speed (interface #5) for correlation. lat/lon is populated for every entry_type now, not just cloud-sourced ones — this is what lets Verification Engine unambiguously associate a jerk event with the correct entry when multiple candidates may be in flight simultaneously, rather than relying on kinematic timing alone.

------------------------------
## 8. Verification Engine → Connectivity Manager (Telemetry)

* Protocol: Internal IPC / MQTT
* Payload: timestamp, pothole_id, event_type (NEW | ACTIVE | CLEARED), lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score
* Rationale: entry_type (interface #7) and jerk confirmation (interface #2) are two independent signals: 
    - LIVE_ONLY + jerk confirms -> NEW; 
    - LIVE_ENRICHED or CLOUD_SUBSTITUTED + jerk confirms -> ACTIVE; 
    - CLOUD_CLEAR + jerk absent (expected) -> CLEARED; 
    - CLOUD_CLEAR + jerk unexpectedly fires -> ACTIVE. 
    - Any other combination — jerk absent for LIVE_ONLY, LIVE_ENRICHED, or CLOUD_SUBSTITUTED — produces no report at all. lat/lon in this report comes directly from the entry received on interface #7, not a separate lookup.

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
