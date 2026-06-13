# C1 Silent-Drift Guards (Device-first) Implementation Plan

## Overview

This plan implements **only the #1 ranked refactor opportunity from the research** — **C1: silent-drift of Device field mapping** — and only its **guard-first** step. The canonical `Device` field set (`Device.java:49-72`) is hand-copied across ~7 silent points (copy-ctor, `updateDevice`, `AbstractDeviceEntity`, proto, EDQS `DeviceFields`/`FieldsUtil`, TS mirror). Adding a field is the most common operation and the least protected: the compiler does not catch a forgotten mapping, which has caused real production drift (`version`, `displayName`, EDQS field sync).

We turn the two highest-value, weakest-covered points into **red tests** using a **reflection-driven** approach so a *new* `Device` field that someone forgets to map fails automatically — no test edit required. We **stop before codegen/registry**: this plan adds (and, where it exposes a genuine bug, fixes) tests. It does not collapse the mapping points into one source.

This is the guard-first first step the research recommends. It is near-zero risk, reversible, and de-risks any future structural work on C1.

## Current State Analysis

- `Device` (`common/data/.../Device.java:45`) implements `HasVersion` and carries: `tenantId, customerId, name, type, label, deviceProfileId, deviceData(+deviceDataBytes), firmwareId, softwareId, externalId, version` plus inherited `id/createdTime/additionalInfo`.
- **Proto round-trip seam already exists.** `ProtoUtilsTest.protoSerializationDeserializationEntities()` (`common/proto/.../ProtoUtilsTest.java:252-297`) builds an `EasyRandom` `Device`, does `toProto → fromProto`, and asserts `isEqualTo`. It covers 8 entities. Because `Device` uses `@EqualsAndHashCode(callSuper = true)`, a field dropped by `toProto`/`fromProto` *should* break equality — but this relies on (a) `EasyRandom` actually populating that field and (b) the field participating in `equals`. There is no explicit assertion that the round-trip exercises the *complete* field set, so a field that `EasyRandom` leaves null/default or that `equals` excludes could drift silently.
- **EDQS path is the weakest-covered.** `FieldsUtil.toFields(Device)` (`common/data/.../edqs/fields/FieldsUtil.java:130-142`) hand-builds `DeviceFields` from `{id, createdTime, customerId, name, type, deviceProfileId, label, additionalInfo, version}`. It **deliberately omits** `tenantId` (EDQS partition key, handled separately), `firmwareId`, `softwareId`, `externalId`, `deviceData/deviceDataBytes`. `DeviceFields` (`DeviceFields.java`) is a projection-subset, not a faithful replica — the guard must preserve this intent, not flag it. There is no test asserting the projection is complete-by-intent.
- **No Java CI.** No PR runs any Java build or test (only two config/license workflows). Every test added here is the de-facto regression net; there is no automatic backstop on merge. Local `mvn test` + review is the gate.

## Desired End State

After this plan:

- Adding a new field to `Device` and forgetting to map it into the **proto** path **fails `ProtoUtilsTest`** with a clear message naming the missing field.
- Adding a new field to `Device` and forgetting to add it to the **EDQS** projection (when it is *not* an intentional omission) **fails a new EDQS parity test**.
- The set of intentionally-omitted EDQS fields is **encoded as an explicit, commented allowlist** — so the deliberate projection-subset is documented in executable form, and removing a field from the allowlist (i.e. deciding it *should* be projected) is a one-line, reviewable change.
- The same guard pattern is **proven and reusable** for the other `HasVersion`/EDQS-mapped entities (Phase 2 applies it).
- A backend compile + `@DaoSqlTest` CI job is **documented and ready to adopt** (Phase 3), but adoption does not gate Phases 1–2.

**Verification:** the new/strengthened tests pass on the current tree (after any genuine drops found are fixed or any deliberate omissions are allowlisted); deliberately deleting a `Device` field mapping locally makes the relevant guard go red.

### Key Discoveries:

- Round-trip guard already exists and is extensible: `ProtoUtilsTest.java:252-297` (`assertEqualDeserializedEntity` helper at `:295`).
- EDQS Device projection and its intentional omissions: `FieldsUtil.java:130-142`, `DeviceFields.java:31-57`.
- Intentional-omission allowlist for Device EDQS parity: `tenantId, firmwareId, softwareId, externalId, deviceData, deviceDataBytes`.
- `FieldsUtil.toFields(Object)` is a 17-branch `instanceof` dispatch (`FieldsUtil.java:44-82`) — the single seam where every entity→Fields conversion is reachable, useful for Phase 2 generalization.

## What We're NOT Doing

- **No codegen / MapStruct / field-descriptor registry.** Explicitly deferred — guards only.
- **No collapsing of the 7 mapping points.** The duplication remains; we only make it loud.
- **No guards for the copy-ctor `Device(Device)`, `updateDevice`, or `AbstractDeviceEntity` DTO↔entity mappings** in this plan. (Persistence is partly covered by the ~272 `@DaoSqlTest` suite; these are lower marginal value than the proto+EDQS points.)
- **No TS-mirror guard** (`device.models.ts`) — cross-language, out of scope.
- **No structural moves** from C2/C3, and **no work on C4/C5/C6** — those are separate, deferred opportunities.
- **CI adoption is not a blocking prerequisite** — Phase 3 is recommended, not gating.

## Implementation Approach

Guard-first, reflection-driven, Device-first. Each phase's first step is a pure test addition (zero production change). The one place production code may change is Phase 1 triage: if a new guard exposes a field that is *genuinely* dropped today, we fix that mapping in the same phase and rely on the manual-verification gate (no CI) before merge. Deliberate omissions are encoded in an allowlist with a comment rather than "fixed".

The reflection approach is the crux: enumerate `Device`'s mappable fields once and assert each is represented, so a future field fails the guard with **no test edit**. An explicit literal list would re-introduce the exact human step that causes the drift, so it is rejected.

## Critical Implementation Details

- **Intentional-omission allowlist is load-bearing.** The EDQS parity test must treat `{tenantId, firmwareId, softwareId, externalId, deviceData, deviceDataBytes}` as expected-absent for `Device`. Without the allowlist the test false-positives on the deliberate projection-subset; the allowlist *is* the executable documentation of that intent. Each entry needs a one-line comment on *why* it is omitted (partition key vs. not-query-projected).
- **Reflection must mirror `equals` semantics, not raw fields.** `deviceData` is `transient` and round-trips via `deviceDataBytes`; the proto-completeness assertion should key off the fields that actually participate in identity/serialization, not blindly every declared field, or it will produce spurious reds. Reuse the existing `EasyRandom`-then-`equals` mechanism as the source of truth and add a completeness assertion that fails loudly when `EasyRandom` leaves a serialization-relevant field unset.
- **No CI means local-run discipline.** Both phases must be verified with local `mvn test` on the affected modules (`common/proto`, `common/data`) before merge; there is no automatic backstop.

## Phase 1: Device silent-drift guards

### Overview

Make the two highest-value Device mapping points (proto round-trip, EDQS projection) fail loudly on a forgotten field, and triage anything the new guards expose.

### Changes Required:

#### 1. Strengthen the proto round-trip completeness guard for Device

**File**: `common/proto/src/test/java/org/thingsboard/server/common/util/ProtoUtilsTest.java`

**Intent**: Ensure the existing `Device` round-trip (`:254-257`) actually exercises the *complete* serialization-relevant field set, so a new `Device` field absent from `ProtoUtils.toProto/fromProto` or `DeviceProto` fails with a message naming the field — rather than passing because `EasyRandom` happened to leave it unset or `equals` excluded it.

**Contract**: Keep the `EasyRandom → toProto → fromProto → isEqualTo` flow as the behavioral check. Add a reflection-backed completeness assertion over `Device`'s serialization-relevant fields (the fields participating in `equals`) confirming each is non-default in the `EasyRandom` instance before the round-trip, so the equality check is known to actually cover them. The failure message must name the offending field. No production change in this item.

#### 2. Add a reflection-driven EDQS `DeviceFields` parity test

**File**: `common/data/src/test/java/org/thingsboard/server/common/data/edqs/fields/` (new test class, e.g. `FieldsUtilDeviceParityTest.java`)

**Intent**: Assert that every `Device` field is *either* present in the `DeviceFields` projection produced by `FieldsUtil.toFields(Device)` *or* listed in an explicit intentional-omission allowlist — so a new, non-omitted field that someone forgets to add to the EDQS projection fails.

**Contract**: Build a representative `Device` (random or fixture), call `FieldsUtil.toFields(device)`, and reflectively compare `Device`'s mappable field names against `DeviceFields`' populated fields. Intentional omissions allowlist (with per-entry comment): `tenantId` (EDQS partition key), `firmwareId`, `softwareId`, `externalId`, `deviceData`/`deviceDataBytes` (not query-projected). A field that is neither projected nor allowlisted fails the test naming the field. The allowlist must be a named constant so removing an entry (deciding to project that field) is a one-line, reviewable change.

#### 3. Triage reds exposed by items 1–2

**File**: (depends on findings) `common/proto/.../ProtoUtils.java`, `common/proto/.../proto/queue.proto`, or `common/data/.../edqs/fields/FieldsUtil.java` / `DeviceFields.java`; or the allowlist constant from item 2.

**Intent**: For each field the new guards flag: if it is a **genuine** silent drop, fix the mapping so the round-trip/projection is correct; if it is a **deliberate** omission, encode it in the allowlist with a comment. Decide per field, fix in this phase — do not leave the guard red and do not blanket-allowlist current state.

**Contract**: Each red resolves to exactly one of: (a) a mapping fix (production change — gated by manual verification below), or (b) an allowlist entry with a justifying comment. No red is left unexplained.

### Success Criteria:

#### Automated Verification:

- `common/proto` tests pass: `mvn -pl common/proto test -Dtest=ProtoUtilsTest`
- New EDQS parity test passes: `mvn -pl common/data test -Dtest=FieldsUtilDeviceParityTest`
- Full affected-module build is green: `mvn -pl common/proto,common/data -am test`

#### Manual Verification:

- Locally delete one `Device→proto` field mapping and confirm `ProtoUtilsTest` goes red naming that field; revert.
- Locally add a throwaway field to `Device` (not allowlisted) and confirm the EDQS parity test goes red; revert.
- If any production mapping was changed in item 3, confirm the affected behavior (proto serialization / EDQS projection) manually for that field and confirm no regression in related Device flows.
- Confirm every allowlist entry has a justifying comment and reads as intentional.

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing (including any production mapping fix from item 3) was successful before proceeding to Phase 2.

---

## Phase 2: Generalize the guard pattern to other HasVersion / EDQS entities

### Overview

Once the pattern is proven on `Device`, extend the same proto-completeness and EDQS-parity guards to the other entities that share the copied mapping shape, closing the named gap the research flagged.

### Changes Required:

#### 1. Extend EDQS parity guards across the entity set

**File**: `common/data/src/test/java/org/thingsboard/server/common/data/edqs/fields/` (parameterize the Phase 1 test or add siblings)

**Intent**: Apply the reflection-driven parity check to the other entities dispatched by `FieldsUtil.toFields` — Asset, Customer, EntityView, Edge, User, Dashboard, RuleChain, RuleNode, WidgetType, WidgetsBundle, DeviceProfile, AssetProfile, Tenant, TenantProfile, QueueStats, ApiUsageState — each with its own intentional-omission allowlist.

**Contract**: Prefer a parameterized test driven by the `FieldsUtil.toFields(Object)` dispatch (`FieldsUtil.java:44-82`) so the entity set stays in sync with the production dispatch. Each entity carries its own allowlist constant; an unmapped, non-allowlisted field fails naming the entity and field. Entities whose projection nuances are genuinely unclear may be deferred with an explicit `// TODO` rather than a wrong allowlist — but log which were deferred (no silent caps).

#### 2. Extend the proto round-trip completeness guard to the other round-tripped entities

**File**: `common/proto/src/test/java/org/thingsboard/server/common/util/ProtoUtilsTest.java`

**Intent**: Apply the Phase 1 completeness assertion to the other 7 entities already in `protoSerializationDeserializationEntities()` so each round-trip is known to cover its full serialization-relevant field set.

**Contract**: Reuse the Phase 1 completeness helper across the existing entities (DeviceCredentials, DeviceProfile, Tenant, TenantProfile, TbResource, ApiUsageState, RepositorySettings). Triage any reds as in Phase 1 item 3.

### Success Criteria:

#### Automated Verification:

- Parameterized EDQS parity tests pass for all covered entities: `mvn -pl common/data test -Dtest=*ParityTest`
- `ProtoUtilsTest` passes with extended coverage: `mvn -pl common/proto test -Dtest=ProtoUtilsTest`
- Affected modules build green: `mvn -pl common/proto,common/data -am test`

#### Manual Verification:

- Spot-check 2–3 entities' allowlists against their `FieldsUtil.toFields` builder to confirm omissions are intentional, not bugs.
- Confirm any deferred entities are explicitly logged/`TODO`-marked, not silently skipped.
- Confirm any production mapping fix triggered by the extended guards is behavior-verified for the affected entity/field.

**Implementation Note**: After completing this phase and all automated verification passes, pause for manual confirmation before treating the C1 guard program as complete.

---

## Phase 3: (Recommended, non-blocking) Backend compile + @DaoSqlTest CI gate

### Overview

The single highest-leverage multiplier for the whole refactor program: a PR job that compiles the backend and runs the `@DaoSqlTest` suite, so guards added here (and any future structural work) actually gate merges. **This phase does not block Phases 1–2** — it is documented and ready to adopt, and can land before, during, or after them.

### Changes Required:

#### 1. Add a backend build+test GitHub workflow

**File**: `.github/workflows/` (new workflow, e.g. `backend-ci.yml`)

**Intent**: On PRs touching backend modules, run a Maven compile and the test suite (at minimum the fast/non-Spring tests added in Phases 1–2; ideally the `@DaoSqlTest` suite) so silent-drift guards become enforced gates rather than local-only checks.

**Contract**: A workflow triggered on `pull_request` that runs the JDK setup matching the project, a Maven build, and the test phase for the affected modules. Surface pre-existing failures honestly — if the full suite is too slow or already red, scope the first iteration to `common/proto` + `common/data` (where the new guards live) and document the narrowing in the workflow and PR description. Do not silently restrict coverage without saying so.

### Success Criteria:

#### Automated Verification:

- Workflow runs on PR and executes the Maven build: visible green/red check on a test PR.
- The Phase 1–2 guard tests run inside the workflow: `ProtoUtilsTest` and the EDQS parity tests appear in the job output.

#### Manual Verification:

- Open a throwaway PR that breaks a guard and confirm the CI check goes red.
- Confirm job runtime is acceptable for PR feedback; if the full `@DaoSqlTest` suite is too slow, confirm the documented narrowed scope is recorded.

**Implementation Note**: Recommended, not required. If adopted, confirm it does not block existing contributor workflows before enabling as a required check.

---

## Testing Strategy

### Unit Tests:

- Proto round-trip completeness for `Device` (Phase 1) and the other 7 round-tripped entities (Phase 2) — fast, no Spring/DB.
- EDQS `FieldsUtil` parity for `Device` (Phase 1) and the broader entity set (Phase 2) — fast, no Spring/DB.
- Key edge cases: intentional omissions (allowlist), `transient`/byte-backed fields (`deviceData`/`deviceDataBytes`), projection-subset semantics.

### Integration Tests:

- None added. Existing `@DaoSqlTest` persistence coverage (~272 tests) remains the backstop for persistence mapping; Phase 3 optionally wires it into CI.

### Manual Testing Steps:

1. Delete a `Device→proto` mapping locally → `ProtoUtilsTest` red naming the field → revert.
2. Add a non-allowlisted throwaway field to `Device` → EDQS parity test red → revert.
3. Remove an entry from the EDQS allowlist → parity test red (proves the allowlist is load-bearing) → revert.
4. If any production mapping was fixed in triage, manually verify that field's proto serialization / EDQS projection behaves correctly.

## Performance Considerations

Tests are pure unit tests (no Spring context, no DB), so they add negligible build time. The Phase 3 CI job's runtime is the only performance concern; if the full `@DaoSqlTest` suite is too slow for PR feedback, scope the first iteration to the guard-bearing modules.

## Migration Notes

None — this plan adds tests and, at most, corrects forgotten field mappings exposed by those tests. No schema, API, or data migration.

## References

- Research: `context/changes/refactor-opportunities/research.md` (ranking #1 = C1; guard seam at `ProtoUtilsTest:253`; EDQS default-off; no-CI feasibility finding)
- Source of candidates: `context/changes/os-encji/research.md`
- Existing round-trip seam: `common/proto/src/test/java/org/thingsboard/server/common/util/ProtoUtilsTest.java:252-297`
- EDQS Device projection: `common/data/src/main/java/org/thingsboard/server/common/data/edqs/fields/FieldsUtil.java:130-142`, `DeviceFields.java:31-57`
- Canonical Device field set + silent copy-ctor/update: `common/data/src/main/java/org/thingsboard/server/common/data/Device.java:49-72,82-111`

## Progress

> Convention: `- [ ]` pending, `- [x]` done. Append ` — <commit sha>` when a step lands. Do not rename step titles. See `references/progress-format.md`.

### Phase 1: Device silent-drift guards

#### Automated

- [ ] 1.1 `common/proto` tests pass: `mvn -pl common/proto test -Dtest=ProtoUtilsTest`
- [ ] 1.2 New EDQS parity test passes: `mvn -pl common/data test -Dtest=FieldsUtilDeviceParityTest`
- [ ] 1.3 Affected-module build green: `mvn -pl common/proto,common/data -am test`

#### Manual

- [ ] 1.4 Deleting a Device→proto mapping makes `ProtoUtilsTest` go red naming the field (then revert)
- [ ] 1.5 Adding a non-allowlisted Device field makes the EDQS parity test go red (then revert)
- [ ] 1.6 Any production mapping changed in triage is behavior-verified with no regression
- [ ] 1.7 Every allowlist entry has a justifying comment and reads as intentional

### Phase 2: Generalize the guard pattern to other HasVersion / EDQS entities

#### Automated

- [ ] 2.1 Parameterized EDQS parity tests pass for all covered entities: `mvn -pl common/data test -Dtest=*ParityTest`
- [ ] 2.2 `ProtoUtilsTest` passes with extended coverage: `mvn -pl common/proto test -Dtest=ProtoUtilsTest`
- [ ] 2.3 Affected modules build green: `mvn -pl common/proto,common/data -am test`

#### Manual

- [ ] 2.4 Spot-check 2–3 entities' allowlists against their `FieldsUtil.toFields` builder
- [ ] 2.5 Any deferred entities are explicitly logged/`TODO`-marked, not silently skipped
- [ ] 2.6 Any production mapping fix from extended guards is behavior-verified

### Phase 3: (Recommended, non-blocking) Backend compile + @DaoSqlTest CI gate

#### Automated

- [ ] 3.1 Workflow runs on PR and executes the Maven build (visible check on a test PR)
- [ ] 3.2 Phase 1–2 guard tests run inside the workflow

#### Manual

- [ ] 3.3 A PR that breaks a guard makes the CI check go red
- [ ] 3.4 Job runtime acceptable; any narrowed scope is documented
