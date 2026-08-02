## 1. Perception System → Sensor Fusion

* Protocol: Automotive Ethernet / PCIe
* Payload: timestamp, bounding_box, est_length_cm, est_width_cm, est_depth_cm, visual_confidence_score
* Rationale: Delivers forward-looking visual object detection, bounding geometry, estimated dimensions, and optical confidence level from camera and LiDAR pipelines.

------------------------------
## 2. Jerk Sensor → Sensor Fusion

* Protocol: CAN-FD
* Payload: timestamp, sensor_health_state, jerk_magnitude, jerk_confidence_score
* Rationale: Delivers reactive physical impact readings, vertical spike magnitudes, and IMU operational status.

------------------------------
## 3. Sensor Fusion → Geotagging & Verification Engine: 

* Protocol: Internal IPC / Shared Memory
* Payload: timestamp, fused_pothole_id, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score
* Rationale: Passes the unified pothole observation and its calculated pothole_confidence_score (fusing visual and IMU feeds) to be geotagged and verified.  

------------------------------
## 4. Vehicle State / Odometry → Geotagging & Verification Engine

* Protocol: CAN-FD
* Payload: timestamp, vehicle_speed_mph
* Rationale: Standardized broadcast from the vehicle's central state
estimator. Crucial for context: a high jerk confidence at 15 mph means
a much deeper pothole than a high jerk confidence at 70 mph.

------------------------------
## 5. Localization Module → Geotagging & Verification Engine

* Protocol: Internal IPC / Automotive Ethernet
* Payload: timestamp, latitude, longitude, altitude, hd_map_lane_id
* Rationale: Supplies pinpoint spatial positioning data at millisecond precision
to accurately anchor detected edge anomalies to geographic coordinates.

------------------------------
## 6. Geotagging & Verification Engine → Central Path Planner
* Protocol: Internal IPC / Shared Memory
* Payload: timestamp, pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score, encounter_speed_mph
* Rationale: Streams real-time, locally confirmed geotagged potholes directly to the Path Planner for immediate edge-level trajectory adjustment or avoidance maneuvers. 

------------------------------
## 7. Geotagging & Verification Engine → Connectivity Manager (Telemetry)
* Protocol: Internal IPC / MQTT
* Payload: timestamp, pothole_id, event_type (NEW | ACTIVE | CLEARED), lat, lon, est_length_cm, est_width_cm, est_depth_cm, pothole_confidence_score, encounter_speed_mph
* Rationale: Offloads geotagged pothole reports and verification status updates (ACTIVE or CLEARED) to the telemetry layer for upstream transmission to the cloud.

------------------------------
## 8. Connectivity Manager → Central Path Planner
* Protocol: Cellular (5G/LTE) / gRPC or REST (cached locally)
* Payload: array[pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm, severity_index, recommended_max_speed]
* Rationale: Streams the cloud-maintained localized list of known upcoming potholes directly to the Central Path Planner to enable proactive, forward look-ahead trajectory planning (lane changes, nudges, deceleration).

------------------------------
## 9. Connectivity Manager → Geotagging & Verification Engine
* Protocol: Internal IPC / Shared Memory
* Payload: array[pothole_id, lat, lon, est_length_cm, est_width_cm, est_depth_cm]
* Rationale: Feeds the cloud's list of expected road conditions into the Verification Engine, allowing real-time edge sensors to verify whether an expected pothole remains ACTIVE or has been patched (CLEARED).  

------------------------------
## 10. Central Path Planner → Vehicle Controls
* Protocol: FlexRay / CAN-FD
* Payload: actuation_timestamp, target_steering_angle, target_acceleration
* Rationale: Transmits the final arbitrated steering and braking commands to drive-by-wire actuators.  