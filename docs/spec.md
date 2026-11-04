# Pit Wall Strategy Console — Baseline Specification

**Version:** 1.0  
**Status:** Approved training baseline  
**Applies from:** Module 6  
**Product:** Pit Wall Strategy Console for the fictional Aurora Grand Prix

## How to use this baseline

Copy this file into the training repository unchanged as `docs/spec.md`. It is the authoritative product specification for Modules 6–16.

Do not modify the fixed product rules, formulas, validation semantics, units, rounding rules, worked results or scope boundaries in this document. If you find a suspected inconsistency, record it in the pull request and request human resolution rather than silently changing the baseline.

Your Module 6 task is to **derive `docs/architecture.md` from this specification**. You decide how the prescribed product should be structured technically; you do not redesign the racing logic.

## 1. Product intent

Apex Velocity Racing currently coordinates strategy decisions across spreadsheets, notes and chat messages. The Pit Wall Strategy Console is a small local web application that gives one fictional race team a traceable place to:

1. capture the current race state;
2. maintain synthetic tyre availability and simplified race events;
3. create single-stop or no-stop candidate strategies;
4. reject invalid candidates with explicit reasons;
5. rank valid candidates with one transparent deterministic model;
6. present a concise strategy brief; and
7. record the strategist’s selected or rejected option and rationale.

The console provides decision support only. It never executes a pit stop, changes live race state automatically or replaces the strategist’s decision.

## 2. Users and core decision

### Race engineer

Captures the latest race snapshot, tyre-set availability and relevant race events.

### Race strategist

Creates and compares candidate strategies, reviews assumptions and makes the final call.

### Technical lead

Owns architecture, agent instructions, quality controls, approvals and traceability.

The product must help answer:

> Which valid next-stop strategy has the lowest projected remaining race time, and why?

## 3. Scope boundaries

### In scope

- One fictional car in one fictional race.
- A 52-lap default Aurora Grand Prix configuration.
- One optional future pit stop per candidate, or no further stop.
- Persistent local race snapshots, tyre sets, race events, candidates, evaluations and decisions.
- Safety Car events as the only race event that affects calculations.
- Deterministic calculations and synthetic, version-controlled data.
- Race Control and Strategy Lab browser views.

### Out of scope

- Live telemetry, real race data and external APIs.
- Multi-car or multi-stop optimisation.
- Weather modelling, tyre-temperature physics or a full-race simulator.
- Machine learning or AI inside the product.
- Autonomous strategy execution.
- Authentication, cloud runtime, real-time updates and websockets.

## 4. Prescribed implementation constraints

The architecture must use:

- Python 3.12;
- FastAPI;
- SQLModel with SQLite;
- Jinja server-rendered HTML with only minimal JavaScript;
- pytest for automated tests; and
- ruff for linting.

All runtime behaviour must work locally without an external service. The deterministic strategy logic must have one source of truth in a pure domain layer and remain independently testable without routes, templates, a browser or a database. Later modules refer to that boundary as `app/domain/strategy.py`.

## 5. Fixed domain contract

All times are measured in seconds. Lap numbers are one-based integers. Tyre age is the number of completed laps on that tyre set and is a non-negative integer.

| Entity | Required fields | Fixed meaning |
|---|---|---|
| Race configuration | `race_id`, `version`, `total_laps`, `base_lap_time_s`, `pit_loss_s`, compound profiles | One immutable, versioned calculation configuration per evaluation. |
| Compound profile | `compound_offset_s`, `degradation_s_per_lap` | Deterministic additions to base lap time. |
| Race snapshot | `id`, `race_config_version`, `tyre_inventory_version`, `race_event_set_version`, `completed_lap`, `position`, `gap_ahead_s`, `gap_behind_s`, `current_tyre_set_id`, `current_tyre_age_laps` | Immutable state immediately after completed lap `L`, pinned to every mutable calculation input. |
| Tyre set | `id`, `tyre_inventory_version`, `compound`, `available` | Synthetic physical set in one immutable inventory version. IDs are unique within that version. Availability is not reserved between candidate comparisons. |
| Race event | `id`, `race_event_set_version`, `type`, `start_lap`, `end_lap`, `note` | Event in one immutable event-set version, with an inclusive lap range. Only `SAFETY_CAR` affects calculation. |
| Candidate strategy | `id`, `snapshot_id`, `race_config_version`, `tyre_inventory_version`, `race_event_set_version`, `calculation_contract_version`, `label`, `pit_lap`, `incoming_tyre_set_id` | Either one planned stop or a no-stop plan, pinned to its complete calculation context. |
| Strategy evaluation | `id`, `candidate_id`, `snapshot_id`, `race_config_version`, `tyre_inventory_version`, `race_event_set_version`, `calculation_contract_version`, `status`, `reason_codes`, `projected_total_s`, `breakdown`, `rank` | Immutable deterministic result for one candidate and one complete calculation context. |
| Decision | `candidate_id`, `disposition`, `rationale` | Human selection or rejection; it has no calculation side effect. |

### Default training seed

Materialise these values later as `data/seed/race-config.json` and `data/seed/tyres.json`:

- Race: `AURORA-52`, configuration version `aurora-v1`, 52 laps, base lap time `90.000 s`, normal pit loss `20.000 s`, Safety Car pit-loss factor `0.60`.
- Soft: offset `-0.400 s`, degradation `0.180 s/lap`.
- Medium: offset `0.000 s`, degradation `0.120 s/lap`.
- Hard: offset `0.350 s`, degradation `0.040 s/lap`.
- Tyre inventory version `aurora-tyres-v1`: M2 Medium currently in use and unavailable for a future selection; H1 Hard available; H2 Hard unavailable; S1 Soft available.
- Race-event set version `aurora-events-v1`: initially empty.

Availability is checked only when a tyre set is selected as an incoming set. Candidate comparisons do not reserve or consume a set.

### Reproducibility and identity

- A persisted candidate must carry all five context references: `snapshot_id`, `race_config_version`, `tyre_inventory_version`, `race_event_set_version` and `calculation_contract_version`.
- Resolve those candidate references exactly. `snapshot_id` must identify an immutable snapshot. The candidate's race-config, tyre-inventory and race-event-set versions must each resolve and equal the corresponding reference on that snapshot. Its calculation-contract version must resolve and equal the loaded baseline's `contract_version`.
- A persisted evaluation must resolve its `candidate_id` and copy all five context references exactly from that candidate. It may not substitute references supplied by a request or by the current environment.
- A missing or mismatched reference is an invalid context. Never fall back to a latest snapshot, latest race configuration, latest tyre inventory, latest event set or default calculation contract.
- A race configuration, tyre inventory, race-event set or snapshot that is already referenced is never edited in place. Any change creates a new immutable version or snapshot.
- Recalculating a candidate creates a new evaluation record; it never silently overwrites the previous result.
- Candidate IDs must be unique within one evaluation set and match `^[A-Z][A-Z0-9_]{0,39}$`.
- Candidate ID is the final deterministic tie-breaker, so duplicates and arbitrary free-text IDs are invalid.

## 6. Fixed lap and stint semantics

Let:

- `L` be the snapshot’s `completed_lap`;
- `P` be a candidate’s `pit_lap`; and
- `T` be `total_laps`.

The following rules are fixed:

1. The snapshot describes the state **after lap `L`**.
2. A planned stop occurs **at the end of lap `P`**.
3. A stopped candidate is valid only when `L < P < T`.
4. “Pit now” means `P = L + 1`.
5. The pit lap belongs to the outgoing stint.
6. For a stopped candidate, the outgoing tyre covers laps `L + 1` through `P`, inclusive.
7. The incoming tyre starts at age zero on lap `P + 1` and covers laps `P + 1` through `T`, inclusive.
8. A no-stop candidate has `pit_lap = null` and `incoming_tyre_set_id = null`; the current tyre covers all laps `L + 1` through `T`.
9. These ranges must cover every remaining lap exactly once. Multi-stop candidates are not supported.

## 7. Fixed projected-time calculation

For each projected lap:

```text
lap_time_s = base_lap_time_s
             + compound_offset_s
             + degradation_s_per_lap × tyre_age_at_start
```

### Tyre age

For an outgoing tyre on projected lap `R`:

```text
tyre_age_at_start = current_tyre_age_laps + (R - (L + 1))
```

For an incoming tyre on projected lap `R`:

```text
tyre_age_at_start = R - (P + 1)
```

The incoming tyre therefore has age zero on lap `P + 1`. Tyre age increments after each completed lap.

### Pit loss

- Add `pit_loss_s` exactly once for a stopped candidate.
- If at least one `SAFETY_CAR` event satisfies `start_lap <= P <= end_lap`, multiply the pit loss by `0.60`.
- Safety Car range endpoints are inclusive.
- Eligibility depends on stop lap `P`, never on incoming-stint start `P + 1`.
- Multiple overlapping Safety Car events do not compound the reduction.
- A no-stop candidate adds no pit loss.

### Total, rounding and ranking

1. Sum all unrounded projected lap times and the applicable pit loss.
2. Round the final total once to three decimal places using decimal `ROUND_HALF_UP`.
3. Rank valid candidates on that rounded total.
4. Sort by lower `projected_total_s`, then lower effective pit lap, then ascending candidate ID.
5. A no-stop candidate’s effective pit lap is `T + 1` for tie-breaking only.
6. Candidate IDs in the fixtures use uppercase ASCII and are compared lexicographically.

Routes, templates and persistence code may display the calculation result but must not reimplement it.

## 8. Fixed validation behaviour

Return every applicable reason code in the following precedence order. Do not stop after the first error.

| Order | Reason code | Condition |
|---:|---|---|
| 1 | `SNAPSHOT_NOT_FOUND` | The candidate's `snapshot_id` is missing or does not resolve to an immutable snapshot. |
| 2 | `RACE_CONFIG_VERSION_NOT_FOUND` | The candidate's `race_config_version` is missing or does not resolve to an immutable race configuration. |
| 3 | `TYRE_INVENTORY_VERSION_NOT_FOUND` | The candidate's `tyre_inventory_version` is missing or does not resolve to an immutable tyre inventory. |
| 4 | `RACE_EVENT_SET_VERSION_NOT_FOUND` | The candidate's `race_event_set_version` is missing or does not resolve to an immutable race-event set. |
| 5 | `CALCULATION_CONTRACT_VERSION_NOT_FOUND` | The candidate's `calculation_contract_version` is missing or does not resolve to an approved calculation contract. |
| 6 | `CANDIDATE_CONTEXT_MISMATCH` | All references resolve, but any candidate config, inventory or event-set version differs from its resolved snapshot reference, or its calculation-contract version differs from the loaded contract's `contract_version`. |
| 7 | `EVALUATION_CONTEXT_MISMATCH` | When an existing evaluation is validated, its `candidate_id` does not resolve or any of its five context references differs from the resolved candidate's reference. New evaluations must copy the candidate references exactly. |
| 8 | `INVALID_TOTAL_LAPS` | `T` is not a positive integer. |
| 9 | `SNAPSHOT_LAP_OUT_OF_RANGE` | `L` is not an integer, `L < 0`, or `L >= T`. Lap zero is allowed as a pre-race snapshot. |
| 10 | `CURRENT_TYRE_SET_NOT_FOUND` | `current_tyre_set_id` does not resolve within the referenced tyre-inventory version. |
| 11 | `INVALID_TYRE_AGE` | `current_tyre_age_laps` is negative or not an integer. |
| 12 | `CANDIDATE_ID_INVALID` | Candidate ID is empty or does not match `^[A-Z][A-Z0-9_]{0,39}$`. |
| 13 | `CANDIDATE_ID_DUPLICATE` | Candidate ID occurs more than once in the evaluation set. |
| 14 | `PIT_LAP_NOT_INTEGER` | `pit_lap` is neither `null` nor an integer. |
| 15 | `STOP_NOT_AFTER_SNAPSHOT` | Integer `pit_lap` satisfies `P <= L`. |
| 16 | `STOP_ON_OR_AFTER_FINAL_LAP` | Integer `pit_lap` satisfies `P >= T`. |
| 17 | `INCOMING_TYRE_REQUIRED` | A stopped candidate has no incoming tyre-set ID. |
| 18 | `INCOMING_TYRE_NOT_FOUND` | The incoming tyre-set ID is unknown within the referenced inventory version. |
| 19 | `TYRE_SET_UNAVAILABLE` | The known incoming tyre set has `available = false`. |
| 20 | `NO_STOP_WITH_INCOMING_TYRE` | A no-stop candidate supplies an incoming tyre-set ID. |
| 21 | `UNKNOWN_COMPOUND` | A required current or known incoming tyre uses a compound with no profile. |

Resolve context without fallback before evaluating scalar or strategy rules. Return every applicable context code in the table order; only test `CANDIDATE_CONTEXT_MISMATCH` after all five candidate references resolve, and only test `EVALUATION_CONTEXT_MISMATCH` when validating an existing evaluation. Context-level failures such as an invalid race configuration, snapshot, inventory, event set or current tyre are copied to every candidate evaluation in that set. Comparisons involving `P` are evaluated only when `pit_lap` is an integer, so validation itself never crashes on malformed input.

An invalid candidate must remain visible with its ordered `reason_codes`, but it receives:

- `projected_total_s = null`;
- `breakdown = null`; and
- `rank = null`.

Invalid candidates are excluded from the valid ranking. The UI must not present a fallback score or silently repair invalid input.

## 9. Required evaluation and brief output

### Strategy evaluation

Every evaluation uses this fixed top-level shape:

- `id`
- `candidate_id`
- `snapshot_id`
- `race_config_version`
- `tyre_inventory_version`
- `race_event_set_version`
- `calculation_contract_version`
- `status`: `VALID` or `INVALID`
- `reason_codes`: ordered array, empty for a valid candidate
- `projected_total_s`: rounded number or `null`
- `breakdown`: object or `null`
- `rank`: one-based integer among valid candidates, or `null`

The `breakdown` object for a valid candidate contains exactly these keys:

- `outgoing_stint`: `start_lap`, `end_lap`, `tyre_set_id`, `compound`, `start_age_laps`, `lap_count`, `subtotal_s`
- `incoming_stint`: the same fields, or `null` for no-stop
- `pit_loss`: `base_s`, `applied_factor`, `applied_s`, `safety_car_event_ids`
- `unrounded_total_s`
- `rounded_total_s`
- `tie_break`: `effective_pit_lap`, `candidate_id`
- `assumptions`: ordered array of `{code, label, value}` objects

Required assumption codes are `SNAPSHOT_ID`, `RACE_CONFIG_VERSION`, `TYRE_INVENTORY_VERSION`, `RACE_EVENT_SET_VERSION`, `CALCULATION_CONTRACT_VERSION`, `SAFETY_CAR_EVENTS` and `TYRE_AVAILABILITY_AS_OF_SNAPSHOT`. The last two must report the exact referenced event-set and inventory versions rather than whatever data is latest when the view opens.

### Ranked strategy brief

The structured input to the rendered Strategy Lab brief contains exactly:

- `top_candidate_id`: winning valid candidate ID, or `null` when no candidate is valid
- `alternatives`: every candidate in display order, each with `candidate_id`, `label`, `pit_lap`, `status`, `rank`, `projected_total_s`, `delta_to_top_s` and `reason_codes`
- `assumptions`: the required labelled assumptions above
- `summary`: deterministic human-readable text of no more than 120 whitespace-separated words

Valid candidates appear first in ranking order, followed by invalid candidates in candidate-ID order. An invalid alternative pairs its ID with all ordered reason codes. The rendered view may style or format these fields but must not infer missing values or change their meaning.

For each valid alternative, calculate `delta_to_top_s` from the already rounded totals: `ROUND_HALF_UP(projected_total_s - top.projected_total_s, 3)`. The winning candidate therefore has `0.000`. Invalid alternatives have `delta_to_top_s = null`. If no candidate is valid, `top_candidate_id = null` and every alternative has `delta_to_top_s = null`.

## 10. Worked reference cases

The worked fixtures below are normative calculation projections, not fully persistable entity examples. They intentionally include only calculation-relevant inputs; omitted descriptive fields such as position, gaps, labels and notes remain required by the entity contract above. Their `expected_evaluations` contain assertion values only, not the full Strategy Evaluation persistence shape.

### Reference A — Safety Car boundary

Inputs:

- `L = 34`, `T = 52`, base lap time `90.000 s`, normal pit loss `20.000 s`.
- Current Medium M2: age 16, offset `0.000 s`, degradation `0.120 s/lap`.
- Incoming Hard H1: age 0, offset `0.350 s`, degradation `0.040 s/lap`.
- Safety Car: laps 36–38 inclusive.

`PIT_NOW`, `P = 35`:

- Medium lap 35: `91.920 s`.
- Hard laps 36–52: `1541.390 s`.
- Normal pit loss: `20.000 s`, because `P = 35` is outside the Safety Car range.
- Projected total: **`1653.310 s`**.

`WAIT_ONE`, `P = 36`:

- Medium laps 35–36: `183.960 s`.
- Hard laps 37–52: `1450.400 s`.
- Safety Car pit loss: `20.000 × 0.60 = 12.000 s`.
- Projected total: **`1646.360 s`**.

Expected order: `WAIT_ONE`, then `PIT_NOW`.

### Reference B — No-stop handling

Inputs:

- `L = 48`, `T = 52`, base lap time `90.000 s`, normal pit loss `20.000 s`.
- Current Medium M2: age 5, offset `0.000 s`, degradation `0.100 s/lap`.
- Incoming Hard H1: offset `0.300 s`, degradation `0.050 s/lap`.
- No race event.

Results:

- `NO_STOP`: Medium laps 49–52, no pit loss = **`362.600 s`**.
- `PIT_49`: Medium lap 49 plus Hard laps 50–52 plus normal pit loss = **`381.550 s`**.

Expected order: `NO_STOP`, then `PIT_49`.

### Reference C — Tie-breaking

With three remaining laps, zero offsets, zero degradation and identical stopped totals of `310.000 s`, candidates with `P = 3` rank before a candidate with `P = 4`. Candidates sharing `P = 3` rank by ID.

Expected order: `ALPHA`, `EARLY`, `ZETA`, `LATE`.

## 11. Product acceptance criteria

The completed product must satisfy all of the following:

- The same input always produces the same validation, breakdown, total and ranking.
- The reference cases above reproduce their prescribed totals and order.
- Invalid candidates remain visible with stable reason codes and no score or rank.
- Race Control persists and retrieves snapshots, tyre sets and events locally.
- Strategy Lab persists candidate drafts and presents evaluations without duplicating domain calculations.
- A strategist can record a selected or rejected candidate with a rationale.
- No action in the product changes real-world race operations.

## 12. Baseline change control

Version 1.0 is immutable for the training journey. Architecture, implementation and ASF customization may not change its formulas, validation semantics, ranking order or expected results.

A proposed fixed-rule change requires a separately named successor specification, a new `calculation_contract_version`, updated fixtures and explicit human approval before implementation. Existing evaluations remain bound to the version that produced them. An ADR by itself does not authorize changing this baseline.

Refactoring the protected strategy implementation is allowed only when all Version 1.0 outputs remain unchanged and the existing fixtures stay green.

## 13. Module 6 architecture assignment

Do not modify the fixed product rules. Use them as constraints and derive `docs/architecture.md` with Claude Code.

Your architecture must cover:

1. architecture drivers and quality attributes derived from this baseline;
2. logical components and their responsibilities;
3. FastAPI routes and request/response boundaries;
4. SQLModel entities, relationships and SQLite ownership;
5. the pure strategy-domain boundary and its public functions;
6. data flow for snapshot capture, candidate validation, ranking and decision recording;
7. configuration and local dev/test/prod separation;
8. validation, error handling and user-visible failure behaviour;
9. unit, integration and browser-test boundaries;
10. traceability from product acceptance criteria to architecture elements;
11. the Module 7 thin slice and Module 12 coupled ranking integration point; and
12. at least two considered architecture alternatives, one rejected decision and its trade-off.

The architecture may refine component names and internal structure, but it must preserve every fixed rule and prescribed technology constraint in this baseline.

## Appendix A — Machine-readable normative contract

The JSON block below is normative and supports automated checking of the reference calculations. If explanatory prose and this block appear inconsistent, stop and request human resolution.

<!-- BASELINE-CONTRACT:BEGIN -->
```json
{
  "contract_version": "1.0.0",
  "learner_change_policy": "fixed",
  "units": {
    "time": "seconds",
    "lap": "one_based_integer",
    "tyre_age": "completed_laps"
  },
  "lap_semantics": {
    "snapshot_state": "after L",
    "stop_timing": "end of P",
    "pit_now": "P = L + 1",
    "pit_lap_owner": "outgoing",
    "incoming_stint_start": "P + 1",
    "no_stop_pit_lap": null
  },
  "formula": {
    "lap_time_s": "base_lap_time_s + compound_offset_s + degradation_s_per_lap * tyre_age_at_start",
    "outgoing_tyre_age": "current_tyre_age_laps + (R - (L + 1))",
    "incoming_tyre_age": "R - (P + 1)"
  },
  "safety_car": {
    "pit_loss_factor": 0.6,
    "eligibility_lap": "P",
    "range_inclusive": true,
    "overlap_compounds": false
  },
  "rounding": {
    "mode": "ROUND_HALF_UP",
    "decimals": 3,
    "ranking_basis": "rounded_total"
  },
  "ranking": {
    "valid_candidates_only": true,
    "tie_breakers": [
      "projected_total_s",
      "effective_pit_lap",
      "candidate_id"
    ],
    "no_stop_effective_pit_lap": "total_laps + 1"
  },
  "invalid_result": {
    "projected_total_s": null,
    "breakdown": null,
    "rank": null
  },
  "calculation_context": {
    "required_references": [
      "snapshot_id",
      "race_config_version",
      "tyre_inventory_version",
      "race_event_set_version",
      "calculation_contract_version"
    ],
    "required_candidate_references": [
      "snapshot_id",
      "race_config_version",
      "tyre_inventory_version",
      "race_event_set_version",
      "calculation_contract_version"
    ],
    "candidate_snapshot_must_resolve": true,
    "candidate_race_config_must_match_snapshot": true,
    "candidate_tyre_inventory_must_match_snapshot": true,
    "candidate_race_event_set_must_match_snapshot": true,
    "candidate_contract_must_match_baseline": true,
    "evaluation_references_must_match_candidate": true,
    "missing_or_mismatched_context_never_falls_back_to_latest": true,
    "referenced_inputs_are_immutable": true,
    "recalculation_creates_new_evaluation": true
  },
  "candidate_identity": {
    "pattern": "^[A-Z][A-Z0-9_]{0,39}$",
    "unique_within_evaluation_set": true,
    "final_tie_breaker": true
  },
  "structured_output": {
    "rank_base": 1,
    "candidate_fields": [
      "id",
      "snapshot_id",
      "race_config_version",
      "tyre_inventory_version",
      "race_event_set_version",
      "calculation_contract_version",
      "label",
      "pit_lap",
      "incoming_tyre_set_id"
    ],
    "evaluation_fields": [
      "id",
      "candidate_id",
      "snapshot_id",
      "race_config_version",
      "tyre_inventory_version",
      "race_event_set_version",
      "calculation_contract_version",
      "status",
      "reason_codes",
      "projected_total_s",
      "breakdown",
      "rank"
    ],
    "breakdown_fields": [
      "outgoing_stint",
      "incoming_stint",
      "pit_loss",
      "unrounded_total_s",
      "rounded_total_s",
      "tie_break",
      "assumptions"
    ],
    "strategy_brief_fields": [
      "top_candidate_id",
      "alternatives",
      "assumptions",
      "summary"
    ],
    "summary_max_words": 120,
    "invalid_candidates_after_valid": true,
    "delta_to_top": {
      "basis": "rounded_projected_total_s",
      "rounding_mode": "ROUND_HALF_UP",
      "decimals": 3,
      "top_candidate_value": 0,
      "invalid_candidate_value": null,
      "no_valid_candidate_value": null
    }
  },
  "change_control": {
    "fixed_rule_change_requires": "new_human_approved_baseline_version",
    "adr_alone_is_sufficient": false,
    "existing_evaluations_keep_original_contract_version": true
  },
  "default_seed": {
    "tyre_inventory_version": "aurora-tyres-v1",
    "race_event_set_version": "aurora-events-v1",
    "race_config": {
      "race_id": "AURORA-52",
      "version": "aurora-v1",
      "total_laps": 52,
      "base_lap_time_s": 90.0,
      "pit_loss_s": 20.0,
      "safety_car_pit_loss_factor": 0.6,
      "compound_profiles": {
        "SOFT": {
          "compound_offset_s": -0.4,
          "degradation_s_per_lap": 0.18
        },
        "MEDIUM": {
          "compound_offset_s": 0.0,
          "degradation_s_per_lap": 0.12
        },
        "HARD": {
          "compound_offset_s": 0.35,
          "degradation_s_per_lap": 0.04
        }
      }
    },
    "tyre_sets": [
      {
        "id": "M2",
        "compound": "MEDIUM",
        "available": false
      },
      {
        "id": "H1",
        "compound": "HARD",
        "available": true
      },
      {
        "id": "H2",
        "compound": "HARD",
        "available": false
      },
      {
        "id": "S1",
        "compound": "SOFT",
        "available": true
      }
    ],
    "race_events": []
  },
  "validation_reason_precedence": [
    "SNAPSHOT_NOT_FOUND",
    "RACE_CONFIG_VERSION_NOT_FOUND",
    "TYRE_INVENTORY_VERSION_NOT_FOUND",
    "RACE_EVENT_SET_VERSION_NOT_FOUND",
    "CALCULATION_CONTRACT_VERSION_NOT_FOUND",
    "CANDIDATE_CONTEXT_MISMATCH",
    "EVALUATION_CONTEXT_MISMATCH",
    "INVALID_TOTAL_LAPS",
    "SNAPSHOT_LAP_OUT_OF_RANGE",
    "CURRENT_TYRE_SET_NOT_FOUND",
    "INVALID_TYRE_AGE",
    "CANDIDATE_ID_INVALID",
    "CANDIDATE_ID_DUPLICATE",
    "PIT_LAP_NOT_INTEGER",
    "STOP_NOT_AFTER_SNAPSHOT",
    "STOP_ON_OR_AFTER_FINAL_LAP",
    "INCOMING_TYRE_REQUIRED",
    "INCOMING_TYRE_NOT_FOUND",
    "TYRE_SET_UNAVAILABLE",
    "NO_STOP_WITH_INCOMING_TYRE",
    "UNKNOWN_COMPOUND"
  ],
  "worked_fixture_schema": {
    "calculation_inputs_only": true,
    "expected_evaluations_are_assertions_only": true,
    "omitted_fields_do_not_override_entity_contract": true
  },
  "worked_fixtures": [
    {
      "id": "safety_car_boundary",
      "tyre_inventory_version": "fixture-safety-car-tyres-v1",
      "race_event_set_version": "fixture-safety-car-events-v1",
      "race_config": {
        "version": "fixture-safety-car-v1",
        "total_laps": 52,
        "base_lap_time_s": 90.0,
        "pit_loss_s": 20.0,
        "compound_profiles": {
          "MEDIUM": {
            "compound_offset_s": 0.0,
            "degradation_s_per_lap": 0.12
          },
          "HARD": {
            "compound_offset_s": 0.35,
            "degradation_s_per_lap": 0.04
          }
        }
      },
      "snapshot": {
        "id": "SNAP_SC_34",
        "race_config_version": "fixture-safety-car-v1",
        "tyre_inventory_version": "fixture-safety-car-tyres-v1",
        "race_event_set_version": "fixture-safety-car-events-v1",
        "completed_lap": 34,
        "current_tyre_set_id": "M2",
        "current_tyre_age_laps": 16
      },
      "tyre_sets": [
        {
          "id": "M2",
          "compound": "MEDIUM",
          "available": true
        },
        {
          "id": "H1",
          "compound": "HARD",
          "available": true
        }
      ],
      "race_events": [
        {
          "id": "SC1",
          "type": "SAFETY_CAR",
          "start_lap": 36,
          "end_lap": 38
        }
      ],
      "candidates": [
        {
          "id": "PIT_NOW",
          "snapshot_id": "SNAP_SC_34",
          "race_config_version": "fixture-safety-car-v1",
          "tyre_inventory_version": "fixture-safety-car-tyres-v1",
          "race_event_set_version": "fixture-safety-car-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": 35,
          "incoming_tyre_set_id": "H1"
        },
        {
          "id": "WAIT_ONE",
          "snapshot_id": "SNAP_SC_34",
          "race_config_version": "fixture-safety-car-v1",
          "tyre_inventory_version": "fixture-safety-car-tyres-v1",
          "race_event_set_version": "fixture-safety-car-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": 36,
          "incoming_tyre_set_id": "H1"
        }
      ],
      "expected_evaluations": [
        {
          "id": "PIT_NOW",
          "projected_total_s": 1653.31
        },
        {
          "id": "WAIT_ONE",
          "projected_total_s": 1646.36
        }
      ],
      "expected_order": [
        "WAIT_ONE",
        "PIT_NOW"
      ]
    },
    {
      "id": "no_stop_baseline",
      "tyre_inventory_version": "fixture-no-stop-tyres-v1",
      "race_event_set_version": "fixture-no-stop-events-v1",
      "race_config": {
        "version": "fixture-no-stop-v1",
        "total_laps": 52,
        "base_lap_time_s": 90.0,
        "pit_loss_s": 20.0,
        "compound_profiles": {
          "MEDIUM": {
            "compound_offset_s": 0.0,
            "degradation_s_per_lap": 0.1
          },
          "HARD": {
            "compound_offset_s": 0.3,
            "degradation_s_per_lap": 0.05
          }
        }
      },
      "snapshot": {
        "id": "SNAP_NS_48",
        "race_config_version": "fixture-no-stop-v1",
        "tyre_inventory_version": "fixture-no-stop-tyres-v1",
        "race_event_set_version": "fixture-no-stop-events-v1",
        "completed_lap": 48,
        "current_tyre_set_id": "M2",
        "current_tyre_age_laps": 5
      },
      "tyre_sets": [
        {
          "id": "M2",
          "compound": "MEDIUM",
          "available": true
        },
        {
          "id": "H1",
          "compound": "HARD",
          "available": true
        }
      ],
      "race_events": [],
      "candidates": [
        {
          "id": "NO_STOP",
          "snapshot_id": "SNAP_NS_48",
          "race_config_version": "fixture-no-stop-v1",
          "tyre_inventory_version": "fixture-no-stop-tyres-v1",
          "race_event_set_version": "fixture-no-stop-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": null,
          "incoming_tyre_set_id": null
        },
        {
          "id": "PIT_49",
          "snapshot_id": "SNAP_NS_48",
          "race_config_version": "fixture-no-stop-v1",
          "tyre_inventory_version": "fixture-no-stop-tyres-v1",
          "race_event_set_version": "fixture-no-stop-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": 49,
          "incoming_tyre_set_id": "H1"
        }
      ],
      "expected_evaluations": [
        {
          "id": "NO_STOP",
          "projected_total_s": 362.6
        },
        {
          "id": "PIT_49",
          "projected_total_s": 381.55
        }
      ],
      "expected_order": [
        "NO_STOP",
        "PIT_49"
      ]
    },
    {
      "id": "deterministic_tie_break",
      "tyre_inventory_version": "fixture-tie-break-tyres-v1",
      "race_event_set_version": "fixture-tie-break-events-v1",
      "race_config": {
        "version": "fixture-tie-break-v1",
        "total_laps": 5,
        "base_lap_time_s": 100.0,
        "pit_loss_s": 10.0,
        "compound_profiles": {
          "MEDIUM": {
            "compound_offset_s": 0.0,
            "degradation_s_per_lap": 0.0
          },
          "HARD": {
            "compound_offset_s": 0.0,
            "degradation_s_per_lap": 0.0
          }
        }
      },
      "snapshot": {
        "id": "SNAP_TIE_2",
        "race_config_version": "fixture-tie-break-v1",
        "tyre_inventory_version": "fixture-tie-break-tyres-v1",
        "race_event_set_version": "fixture-tie-break-events-v1",
        "completed_lap": 2,
        "current_tyre_set_id": "M2",
        "current_tyre_age_laps": 3
      },
      "tyre_sets": [
        {
          "id": "M2",
          "compound": "MEDIUM",
          "available": true
        },
        {
          "id": "H1",
          "compound": "HARD",
          "available": true
        },
        {
          "id": "H2",
          "compound": "HARD",
          "available": true
        },
        {
          "id": "H3",
          "compound": "HARD",
          "available": true
        }
      ],
      "race_events": [],
      "candidates": [
        {
          "id": "ALPHA",
          "snapshot_id": "SNAP_TIE_2",
          "race_config_version": "fixture-tie-break-v1",
          "tyre_inventory_version": "fixture-tie-break-tyres-v1",
          "race_event_set_version": "fixture-tie-break-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": 3,
          "incoming_tyre_set_id": "H1"
        },
        {
          "id": "EARLY",
          "snapshot_id": "SNAP_TIE_2",
          "race_config_version": "fixture-tie-break-v1",
          "tyre_inventory_version": "fixture-tie-break-tyres-v1",
          "race_event_set_version": "fixture-tie-break-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": 3,
          "incoming_tyre_set_id": "H2"
        },
        {
          "id": "ZETA",
          "snapshot_id": "SNAP_TIE_2",
          "race_config_version": "fixture-tie-break-v1",
          "tyre_inventory_version": "fixture-tie-break-tyres-v1",
          "race_event_set_version": "fixture-tie-break-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": 3,
          "incoming_tyre_set_id": "H3"
        },
        {
          "id": "LATE",
          "snapshot_id": "SNAP_TIE_2",
          "race_config_version": "fixture-tie-break-v1",
          "tyre_inventory_version": "fixture-tie-break-tyres-v1",
          "race_event_set_version": "fixture-tie-break-events-v1",
          "calculation_contract_version": "1.0.0",
          "pit_lap": 4,
          "incoming_tyre_set_id": "H1"
        }
      ],
      "expected_evaluations": [
        {
          "id": "ALPHA",
          "projected_total_s": 310.0
        },
        {
          "id": "EARLY",
          "projected_total_s": 310.0
        },
        {
          "id": "ZETA",
          "projected_total_s": 310.0
        },
        {
          "id": "LATE",
          "projected_total_s": 310.0
        }
      ],
      "expected_order": [
        "ALPHA",
        "EARLY",
        "ZETA",
        "LATE"
      ]
    }
  ]
}
```
<!-- BASELINE-CONTRACT:END -->
