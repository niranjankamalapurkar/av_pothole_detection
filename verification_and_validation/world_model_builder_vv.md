# World Model Builder — Verification & Validation Plan (Pothole/Road-Surface Feature)

## Scope

Test cases for this feature's contribution to World Model Builder, tracing to world_model_builder_requirements.md (WMB-REQ) and world_model_builder_non_functional_requirements.md (WMB-NFR). Organized under the same lettered sections as the requirements document for direct cross-reference. A requirement may be covered by more than one test — this is expected, not redundant, where each test isolates a different trigger or edge condition for the same requirement.

**Format per test:** Test ID | Covers | Setup/Fault injection | Expected result.

---

## A. Georeferencing

**WMB-VV-01** — Prior-layer lat/lon → grid conversion correctness
*Covers:* WMB-REQ-01
*Setup:* Inject a cloud-advisory entry at a known `patch_center_lat`/`patch_center_lon`, computed at a known offset and bearing from a fixed, known ego position and heading (e.g., 30 m directly ahead). Repeat at several different bearings relative to heading (ahead, left-ahead, right-ahead), not only due-ahead.
*Expected:* The entry is placed in the grid cell corresponding to its true relative position, within the tolerance of `grid_resolution_cm`, at every tested bearing — confirming the projection/transform is correct generally, not only for the trivial straight-ahead case.

---

## B. Nominal fusion

**WMB-VV-02** — Cloud and live match (co-located detection)
*Covers:* WMB-REQ-02
*Setup:* Inject a valid Live-layer candidate and a valid Prior-layer entry at the same grid location in the same cycle.
*Expected:* Fused output at that cell reflects both sources combined — fused confidence is higher than either source's confidence alone (two independent corroborating sources should increase certainty).

**WMB-VV-03** — Cloud has no entry, Live finds a new candidate
*Covers:* WMB-REQ-02, WMB-REQ-09
*Setup:* Prior layer valid and available overall, but has no entry at the location where Live reports a candidate.
*Expected:* Fused output reflects the Live-only detection, at Live's own confidence — not artificially reduced for lacking Prior corroboration. Distinct from WMB-REQ-03's degenerate case: here Prior is functioning normally, it simply has nothing to say about this specific cell.

**WMB-VV-04** — Cloud has an entry, no Live detection at that cell
*Covers:* WMB-REQ-02
*Setup:* Live layer valid and healthy overall, but reports no candidate at the specific location where Prior has an entry.
*Expected:* Fused output reflects the Prior-only content for that cell. Note for downstream: this is exactly the case POR-REQ-07 (pothole_observation_reporter_functional_requirements.md) depends on being able to detect and suppress at the Reporter — that detection is not this test's concern, but the fused output this test produces is what POR-REQ-07's own test would need to consume, once fmea.md Finding 9's provenance signal exists.

---

## C. Degenerate input handling — data availability

**WMB-VV-05** — Live layer unavailable (health-state trigger), Prior valid
*Covers:* WMB-REQ-06 (first trigger), WMB-REQ-03 (Prior validity, held constant to isolate the Live-side behavior)
*Setup:* Force `perception_health_state` to DEGRADED or FAULTED, or force an empty candidate array, while the Prior layer remains valid and positionally relevant.
*Expected:* Fused output equals Prior-layer content only. Verify internally that this state is distinguishable from "Live confirmed nothing here" — not merely that the output looks the same, per fmea.md Finding 3's concern about that exact ambiguity.

**WMB-VV-06** — Live layer stale (transit-delay trigger), Prior valid
*Covers:* WMB-REQ-06 (second trigger), WMB-NFR-06
*Setup:* `perception_health_state` = NOMINAL, candidate array present and well-formed, but the entry's timestamp is artificially delayed beyond 250 ms.
*Expected:* Same Prior-only fallback as WMB-VV-05, but triggered purely by the deadline check — verify this fires even when `perception_health_state` reports healthy, confirming the deadline-supervision mechanism is genuinely independent of self-reported health, not redundant with it.

**WMB-VV-07** — Prior layer unavailable (no communication), Live valid
*Covers:* WMB-REQ-03 (no-communication trigger)
*Setup:* Connectivity Manager provides no Prior-layer data for a cycle; Live layer valid.
*Expected:* Fused output equals Live-layer content only, not treated as "nothing expected here."

**WMB-VV-08** — Prior layer data invalid (malformed/corrupted), Live valid
*Covers:* WMB-REQ-03 (invalid-data trigger, distinct from no-communication)
*Setup:* Prior-layer data is received but fails validation (structurally malformed, checksum failure) — distinct from simply being absent (WMB-VV-07).
*Expected:* Same Live-only fallback as WMB-VV-07, confirming this second, distinct trigger path is also correctly handled — existence of data and validity of data are different failure classes and should both be tested, not assumed to share one code path by inspection alone.

**WMB-VV-09** — Both layers invalid simultaneously
*Covers:* WMB-REQ-07
*Setup:* Combine WMB-VV-05's Live-unavailable trigger with WMB-VV-07's Prior-unavailable trigger in the same cycle.
*Expected:* No pothole/road-surface fused output is produced for the affected cells this cycle at all — not a fallback to either source, not a "confirmed nothing," genuinely absent from the world model. This is the one case your original list's individual tests do not exercise, since it isn't simply their union — WMB-REQ-07 specifies a third, distinct behavior.

**WMB-VV-10** — Prior-layer already-passed exclusion
*Covers:* WMB-NFR-05
*Setup:* Inject a Prior-layer entry whose position, relative to current ego position and heading, lies behind the vehicle's direction of travel. Repeat at several headings, and once with the entry positioned near-perpendicular to heading (dot product near zero) to check boundary handling.
*Expected:* The entry is excluded from fusion at every tested heading; the near-perpendicular boundary case resolves consistently (not flapping between included/excluded on repeated evaluation of the same input).

**WMB-VV-11** — Localization invalid, Prior otherwise valid
*Covers:* WMB-REQ-05
*Setup:* Inject an invalid/unavailable Localization signal (or confidence below threshold) while Prior-layer data is itself valid and well-formed.
*Expected:* Two things verified together, not just one: Prior layer is excluded from fusion (regardless of the Prior data's own validity), **and** Live layer continues fusing normally and is not incorrectly affected by the same trigger — confirming the exclusion is scoped to exactly the layer that needs localization to be placed.

---

## D. Degenerate input handling — resource availability

**WMB-VV-12** — Thermal throttling / reduced clock frequency
*Covers:* WMB-REQ-08
*Setup:* Simulate a thermal-derating or reduced-clock condition on World Model Builder's own compute, while Prior-layer data is demonstrably valid (not corrupted, not stale, not positionally excluded).
*Expected:* Prior layer is excluded from fusion, Live-only fusion continues — verify this fires even with good Prior data, confirming the trigger is genuinely resource-based and not accidentally coupled to Prior data quality (which would make this indistinguishable from WMB-VV-07/08).

---

## E. Multi-candidate representation

**WMB-VV-13** — Multiple simultaneous, non-overlapping anomalies
*Covers:* WMB-REQ-09
*Setup:* Inject two or more distinct Live-layer candidates and/or Prior-layer entries at different, non-overlapping grid locations in the same cycle.
*Expected:* Each is independently and correctly represented at its own location, with no cross-contamination between them (candidate A's depth values do not appear in candidate B's cells), and no evidence that any object-identity or candidate-pairing step executed (cross-check against WMB-VV-16's complexity-scaling result — if execution time scales with the *product* of candidate counts rather than the *sum*, that's evidence such a step exists despite the architecture description).

---

## F. Cloud data synchronization

**WMB-VV-14** — Prior layer persists unchanged between event-driven updates
*Covers:* WMB-REQ-10
*Setup:* Hold Prior-layer content fixed (no new Connectivity Manager update) across several consecutive Live-layer update cycles.
*Expected:* The same Prior-layer content is used in fusion, unchanged, across all those cycles — not re-fetched, not reset or cleared between cycles — while the Live layer continues updating normally every cycle.

---

## G. Non-functional / timing

**WMB-VV-15** — Fusion WCET compliance
*Covers:* WMB-NFR-01
*Setup:* Profile total fusion execution time under worst-case K (simultaneous Live candidates), L (Prior-layer list size), and N×M (patch dimensions).
*Expected:* Total time ≤ World Model Builder's externally-supplied WCET budget. The budget value itself is external to this feature (WMB-NFR-01's own open item) — this test's pass/fail criterion depends on that value being supplied, not on this feature asserting one.

**WMB-VV-16** — Computational complexity scaling
*Covers:* WMB-NFR-02
*Setup:* Measure execution time across a matrix of K and L values (e.g., K ∈ {1, 5, 10}, L ∈ {1, 10, 50}), holding N×M fixed.
*Expected:* Execution time scales linearly with K and L (confirmed via curve-fit against the measured data), not multiplicatively (K×L) — a multiplicative result would indicate object-matching logic was reintroduced somewhere in the implementation, silently violating WMB-REQ-09 even if the interface contract still looks correct.

**WMB-VV-17** — Inbound Prior-layer bandwidth
*Covers:* WMB-NFR-03
*Setup:* Measure IPC/shared-memory transport time for a worst-case (large L) Prior-layer list.
*Expected:* Transport time is measured and accounted for separately from compute time — confirming it doesn't independently exceed the overall budget verified in WMB-VV-15 even when compute time alone appears within budget.

**WMB-VV-18** — Sustained fusion cycle rate
*Covers:* WMB-NFR-04
*Setup:* Run fusion continuously against a Live layer updating at its documented rate (10–30 Hz) for an extended duration, not a single-shot measurement.
*Expected:* No missed cycles, no growing backlog — confirming WMB-VV-15/16's per-cycle timing bounds actually hold up under sustained real-time operation, not just in isolation.

---

## Requirement-to-test traceability matrix

| Requirement | Covering test(s) |
|---|---|
| WMB-REQ-01 | WMB-VV-01 |
| WMB-REQ-02 | WMB-VV-02, WMB-VV-03, WMB-VV-04 |
| WMB-REQ-03 | WMB-VV-07, WMB-VV-08 (indirectly WMB-VV-09) |
| WMB-REQ-04 | Not independently testable by this feature — an assumption about Localization's own dead-reckoning implementation (world_model_builder_requirements.md), verified as part of Localization's own V&V, not this document's. |
| WMB-REQ-05 | WMB-VV-11 |
| WMB-REQ-06 | WMB-VV-05, WMB-VV-06 (indirectly WMB-VV-09) |
| WMB-REQ-07 | WMB-VV-09 |
| WMB-REQ-08 | WMB-VV-12 |
| WMB-REQ-09 | WMB-VV-13 (cross-checked by WMB-VV-16) |
| WMB-REQ-10 | WMB-VV-14 |
| WMB-NFR-01 | WMB-VV-15 |
| WMB-NFR-02 | WMB-VV-16 |
| WMB-NFR-03 | WMB-VV-17 |
| WMB-NFR-04 | WMB-VV-18 |
| WMB-NFR-05 | WMB-VV-10 |
| WMB-NFR-06 | WMB-VV-06 |

Every requirement has at least one covering test except WMB-REQ-04, which is explicitly out of scope for this feature's own verification by design (it's a dependency assumption, not a requirement this feature implements).