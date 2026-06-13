# C1 Silent-Drift Guards (Device-first) — Plan Brief

> Full plan: `context/changes/refactor-opportunities/plan.md`
> Research: `context/changes/refactor-opportunities/research.md`

## What & Why

The `Device` field set is hand-copied across ~7 silent mapping points (copy-ctor, `updateDevice`, `AbstractDeviceEntity`, proto, EDQS `DeviceFields`/`FieldsUtil`, TS mirror). The compiler does not catch a forgotten field, and that has caused real production drift (`version`, `displayName`, EDQS sync). This plan implements the research's **#1 ranked** opportunity (**C1**) and only its **guard-first** step: make the two highest-value mapping points fail loudly — "silently lost" becomes "red test" — at near-zero risk.

## Starting Point

A proto round-trip seam already exists (`ProtoUtilsTest.java:252-297`: `EasyRandom → toProto → fromProto → equals`, 8 entities) but never asserts it covers the *complete* field set. The EDQS path is weakest-covered: `FieldsUtil.toFields(Device)` (`FieldsUtil.java:130-142`) builds a deliberate projection-subset with no parity test. Critically, **there is no Java CI** — no PR runs any backend build or test, so every test added here is the de-facto regression net.

## Desired End State

Forgetting to map a new `Device` field into the proto path fails `ProtoUtilsTest` naming the field; forgetting the EDQS projection (when not an intentional omission) fails a new reflection-driven parity test. The intentional-omission set (`tenantId, firmwareId, softwareId, externalId, deviceData/deviceDataBytes`) is encoded as a commented allowlist — executable documentation of the projection-subset intent. The pattern is then generalized across the other `HasVersion`/EDQS entities.

## Key Decisions Made

| Decision | Choice | Why (1 sentence) | Source |
| --- | --- | --- | --- |
| Scope of this plan | Just #1 (C1) guard-first | Highest debt-cost, lowest risk; first step is pure test addition | Plan |
| C1 depth | Guards only — stop before codegen | Captures ~all the protection at near-zero risk; codegen is a later, opinionated change | Research / Plan |
| Which mapping points | Proto round-trip completeness + EDQS `FieldsUtil` parity | The two points with documented production incidents and weakest cover | Research / Plan |
| Drift-detection mechanism | Reflection-driven field enumeration + allowlist | A new field fails the guard with no test edit — the whole point of guarding drift | Plan |
| Existing reds | Investigate each: fix real drops, allowlist deliberate omissions | Guard delivers immediate value and the allowlist captures intent durably | Plan |
| Guard reach | Device first (P1), generalize as a follow-up phase (P2) | Prove the pattern before paying the breadth cost | Plan |
| Backend CI job | Recommend (P3), don't block P1–P2 | Highest-leverage multiplier, but guard tests are the gate | Research / Plan |

## Scope

**In scope:** Reflection-driven proto round-trip completeness + EDQS parity guards for `Device` (P1); generalize to other `HasVersion`/EDQS entities (P2); fix any genuine drops the guards expose; documented backend CI job (P3, non-blocking).

**Out of scope:** Codegen/MapStruct/field-registry; collapsing the mapping points; guards for copy-ctor/`updateDevice`/`AbstractDeviceEntity`/TS-mirror; C2/C3/C4/C5/C6; making CI a blocking prerequisite.

## Architecture / Approach

Guard-first, reflection-driven, Device-first. Each phase's first step is a pure test addition (zero production change). The only production code that may change is Phase 1 triage — fixing a field that is *genuinely* dropped today — gated by manual verification since there is no CI. Reflection (not an explicit field list) is essential: it makes a future field fail the guard automatically, instead of re-introducing the human step that causes the drift.

## Phases at a Glance

| Phase | What it delivers | Key risk |
| --- | --- | --- |
| 1. Device guards | Proto-completeness + EDQS parity guards for `Device`, reds triaged | A guard exposes a real existing drop → in-phase production fix widens blast radius (gated by manual verify) |
| 2. Generalize | Same guards across the other `HasVersion`/EDQS entities | Per-entity projection nuances → wrong allowlist; mitigate by deferring unclear entities with explicit `TODO` |
| 3. Backend CI (recommended) | PR job: compile + `@DaoSqlTest` | Full suite may be slow/red → scope first iteration to guard-bearing modules, document narrowing |

**Prerequisites:** Local Maven build of `common/proto` + `common/data`. No CI dependency for P1–P2.
**Estimated effort:** ~1–2 sessions for P1–P2 (pure unit tests + triage); P3 is a small but independently-scoped infra task.

## Open Risks & Assumptions

- **No CI safety net** — local `mvn test` + review is the only gate for P1–P2; discipline required.
- The proto-completeness assertion must mirror `equals`/serialization semantics (e.g. `transient deviceData` ↔ `deviceDataBytes`) or it produces spurious reds.
- Phase 2 assumes each entity's intentional omissions can be determined from its `FieldsUtil.toFields` builder; unclear ones are deferred, not guessed.
- Guards beyond the 7 confirmed Device mapping points (edge sync, import/export DTO) are not enumerated — a known `[unknown]` from the research.

## Success Criteria (Summary)

- A forgotten `Device` field mapping (proto or EDQS) is caught by a red test naming the field, with no test edit needed for the new field.
- The deliberate EDQS projection-subset is documented as a commented, load-bearing allowlist.
- The guard pattern is proven on `Device` and applied across the other `HasVersion`/EDQS entities.
