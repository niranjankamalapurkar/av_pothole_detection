# Autonomous Vehicle Pothole Detection and Avoidance System

## 1. Motivation (The "Why")

To achieve true Level-4/Level-5 autonomy, an Autonomous Vehicle (AV) must continuously demonstrate capabilities that greatly exceed baseline human driving. While modern AV stacks excel at dynamic obstacle avoidance and traffic rule adherence, they must also master the nuanced, environmental adaptations that experienced human drivers perform instinctively—such as adjusting to degraded road surfaces.

**The Business & Experience Impact:**
- **Rider Experience:** If an AV strictly adheres to a 45 mph speed limit on a pothole-riddled road without contextual awareness, the resulting jerks and vibrations severely degrade passenger comfort and trust in the system. Owning the ultimate business outcome means ensuring a smooth, premium ride experience regardless of static road conditions.

- **Fleet Maintenance & Hardware Integrity:** Repeated high-speed impacts with road anomalies cause accelerated wear and tear on suspension systems, tires, and sensitive sensor arrays. Proactive avoidance directly reduces fleet maintenance costs, limits vehicle downtime, and prevents sudden mechanical failures.

- **Shared Fleet Intelligence (The Big Picture):** A single vehicle encountering an unmapped pothole is an isolated physical event. However, without a mechanism to communicate this data, every subsequent vehicle in the network will suffer the exact same degraded experience. The system must close this loop, allowing the entire fleet to learn from a single vehicle's encounter.

## 2. Problem Statement (The "What")

Current autonomous driving systems frequently lack a comprehensive, closed-loop mechanism to identify, communicate, and proactively mitigate the impact of static road surface anomalies. 

**Core Objective:**
This repository proposed an end-to-end Pothole Detection and Mitigation system that allows the AV to intelligently map and navigate road surface irregularities. The system must fulfill a dual mandate:

1. **Edge Detection & Upstream Reporting:** The vehicle perception stack must accurately detect potholes in its path in real-time, assess their severity, and report these geographic coordinates back to a centralized fleet server.

2. **Cloud-Informed Downstream Actuation:** The vehicle must continuously fetch localized, forward-looking road-condition data from the server. Using this shared intelligence, the AV's planning and control systems must execute safe, preemptive driving decisions—such as executing a lane change, nudging within the lane, or initiating smooth deceleration—to neutralize the impact on the passenger before the vehicle reaches the anomaly.