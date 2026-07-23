## 1. Perception System → Sensor Fusion

* Protocol: Automotive Ethernet / PCIe
* Payload: timestamp, bounding_box, est_depth_cm, visual_confidence_score
* Rationale: Provides the predictive visual data. High bandwidth required
for point clouds or high-resolution image bounding.

------------------------------
## 2. Jerk Sensor → Sensor Fusion

* Protocol: CAN-FD
* Payload: timestamp, sensor_health_state, jerk_magnitude
* Rationale: Sensor health state allows the fusion module to easily weight
the IMU's input.

------------------------------
## 3. Vehicle State / Odometry → Geotagger

* Protocol: CAN-FD
* Payload: timestamp, vehicle_speed_mph, steering_angle_deg
* Rationale: Standardized broadcast from the vehicle's central state
estimator. Crucial for context: a high jerk confidence at 15 mph means
a much deeper pothole than a high jerk confidence at 70 mph.

------------------------------
## 4. Localization Module → Geotagger

* Protocol: Internal IPC / Automotive Ethernet
* Payload: timestamp, latitude, longitude, altitude, hd_map_lane_id
* Rationale: Supplies pinpoint spatial positioning data at millisecond precision
to accurately anchor detected edge anomalies to geographic coordinates.

------------------------------
## 5. Geotagger → Connectivety Manager (Telemetry) → Cloud API Gateway

* Protocol: Cellular (5G/LTE) / MQTT
* Payload: lat, lon, est_depth_cm, jerk_confidence_level, encounter_speed_mph, timestamp
* Rationale: Bundles the pothold location with its estimated depth with
the exact speed it was encountered at , allowing the cloud to calculate
safe passage speeds for subsequent vehicles.

------------------------------
## 6. Cloud API Gateway → Central Path Planner

* Protocol: Cellular (5G/LTE) / REST or gRPC
* Payload: array[pothole_id, lat, lon, severity_index, recommended_max_speed]
* Rationale: Delivers a localized cache of upcoming anomalies so the
vehicle can plan its routing proactively rather than reacting blindly
at highway speeds.

------------------------------
## 7. Central Path Planner → Verification Node

* Protocol: Internal IPC / Shared Memory
* Payload: expected_pothole_id, target_lat, target_lon, estimated_arrival_timestamp
* Rationale: Informs the edge audit node of cloud-expected anomalies along the vehicle's trajectory so real-time sensors can verify whether the pothole remains active or has been repaired.

------------------------------
## 8. Verification Node → Connectivety Manager (Telemetry) → Cloud API Gateway
* Protocol: Cellular (5G/LTE) / MQTT
* Payload: pothole_id, verification_status (ACTIVE | CLEARED), observed_confidence, timestamp
* Rationale: Feeds the cloud map healing engine. Confirms persistent road defects or flags repaired/cleared locations to maintain database accuracy.

------------------------------
## 9. Central Path Planner → Vehicle Controls

* Protocol: FlexRay / CAN-FD
* Payload: target_steering_angle, target_acceleration, actuation_timestamp
* Rationale: The final arbitrated, safety-checked trajectory commands to drive-by-wire steering and braking actuators.
