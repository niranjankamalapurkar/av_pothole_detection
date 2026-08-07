## Scope

SOTIF (ISO 21448) analysis is performed on the three items identified in `hara.md` Section 8 as performance-limitation concerns rather than ISO 26262 malfunctions: Ghost Pothole (#2), Single-Vehicle Sensor Miss (#4), and Localization Drift (#7). SOTIF items are not ASIL-classified — there is no malfunction to rate. Instead, each item gets a defined risk acceptance criterion and a verification & validation strategy, argued toward an acceptable residual risk rather than a pass/fail gate.

---

## Method

ISO 21448 organizes scenarios into four areas:

    Area 1 - Known Safe:    identified scenarios, performance is adequate. No action.
    Area 2 - Known Unsafe:  identified scenarios, performance is insufficient. Must be
                            reduced (design change) or argued to an acceptable residual risk.
    Area 3 - Unknown Unsafe: not yet identified scenarios where performance would be
                            insufficient. Found through exposure: field data, simulation,
                            fleet operation — not through brainstorming alone.
    Area 4 - Unknown Safe:  not yet identified scenarios that happen to be fine. Not a
                            safety concern by definition.

The SOTIF process for each item below follows eight steps:

1. Function & Specification - what the function is supposed to do under nominal (non-faulty) operation.
2. Triggering Conditions - environmental/scenario conditions under which that spec could produce insufficient performance. Can be brainstormed directly (as below) or derived more systematically using STPA (System-Theoretic Process Analysis) — not part of ISO 21448 itself, but widely used alongside SOTIF in practice to identify unsafe control actions and generate test scenarios.
3. Vehicle-Level Hazardous Behavior - the unsafe vehicle outcome, traced the same way as a HARA hazardous event.
4. Known/Unknown Categorization - sort findings into Area 1-4; be explicit about what hasn't been found yet.
5. Risk Acceptance Criteria - an explicit, arguable target for "safe enough" (no ASIL exists here, so this must be stated directly).
6. Risk-Reduction Measures - design/algorithm-level options (thresholds, temporal confirmation, response tiering) — not architectural safety mechanisms.
7. Verification & Validation Strategy - how evidence will be generated: scenario testing, simulation, field data.
8. Residual Risk Argument - a cumulative safety case, not a single gate.

---