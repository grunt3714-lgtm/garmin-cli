# Add Workout CRUD and Builder Commands (garmin-cli)

## Context & Problem
- We now have a local OpenAPI spec for Garmin's workout service in `docs/garmin-workout-api.openapi.yaml`.
- The current CLI has no workout-management commands. It exposes activities, health, devices, profile, and sync, but not workout CRUD.
- The current auth/client stack is already aligned with the `connectapi` bearer-token path, which is the right transport for the confirmed workout endpoints.
- The difficult part is not just CRUD. It is authoring valid nested workout steps, repeat groups, targets, and optional zone metadata from a CLI without producing a fragile UX.

## Current State (as of 2026-03-21)
### Confirmed
- `garmin-cli` authenticates via Garmin SSO -> OAuth1 -> OAuth2 bearer token and sends authenticated API requests to `https://connectapi.{domain}`.
- The local OpenAPI spec models workout CRUD, scheduling, and type-discovery endpoints under `/workout-service/...`.
- The Python unofficial client confirms these workout endpoints on `connectapi`:
  - list workouts
  - get workout by id
  - create/upload workout
  - download workout FIT
  - schedule workout
- The current Rust API client supports:
  - authenticated `GET`
  - authenticated JSON `POST`
  - binary download
  - multipart upload
- The current Rust API client does not yet support:
  - authenticated JSON `PUT`
  - authenticated `DELETE`

### Confirmed auth boundary
- The current auth method is correct for the `connectapi` bearer-auth workout surface.
- The current auth method is not a web-session auth flow and should not be treated as compatible with `https://connect.garmin.com/gc-api` cookie-based routes.

### Confirmed modeling gap
- The OpenAPI spec contains richer workout structure than the CLI currently models:
  - nested `RepeatGroupDTO`
  - `ExecutableStepDTO`
  - target-specific fields such as `targetValueOne`, `targetValueTwo`, and `zoneNumber`
  - type-discovery endpoints for workout types and sport types

## Key Takeaways
- We should build workout support against `connectapi`, not against the web proxy.
- CRUD alone is insufficient. The product surface needs a builder workflow that makes nested step authoring practical.
- A file-first workflow is the right MVP. Deeply nested step trees are awkward as pure shell flags.
- Runtime type discovery should be treated as authoritative for target types and related enums.

## Target Types: What We Know vs What Is Still Uncertain
### Confirmed in local OpenAPI spec
The spec currently lists these target type keys:
- `no.target`
- `power.zone`
- `cadence.zone`
- `heart.rate.zone`
- `speed.zone`
- `pace.zone`
- `swim.css.offset`
- `swim.instruction`

### Confirmed field semantics in the spec
- `power.zone`, `cadence.zone`, and `speed.zone` use `targetValueOne` and `targetValueTwo`.
- `pace.zone` uses `targetValueOne` as the faster bound and `targetValueTwo` as the slower bound, both in m/s.
- `heart.rate.zone` uses `zoneNumber`.
- `no.target` does not require target values.

### Known inconsistency
- The Python typed helper code has a `TargetType.OPEN = 6` comment, while the OpenAPI spec currently says id `6` is `pace.zone`.
- That means I was not redacting target types for brevity. We do know a concrete list from the spec, but we do not yet have a single authoritative source that resolves every numeric ID and key mapping cleanly.

### Implementation rule
- Do not treat the Python helper enum as canonical.
- Do not hardcode target type IDs from one reverse-engineered source unless they are validated.
- Use `/workout-service/workout/types` as the source of truth for:
  - `targetTypes`
  - `stepTypes`
  - `conditionTypes`
  - `strokeTypes`
  - `equipmentTypes`
- Prefer key-based internal logic where possible, and use numeric IDs only after we capture and fixture live responses.

## Goals
1. Add a first-class `workouts` CLI area.
2. Support workout CRUD against `connectapi`.
3. Support workout scheduling and unscheduling.
4. Provide a practical CLI workflow for constructing workouts with steps and repeats.
5. Make target-type handling explicit, testable, and grounded in live type-discovery responses.
6. Avoid locking the CLI into an incorrect or outdated step/target enum mapping.

## Non-Goals
- Building a web-proxy cookie auth path.
- Designing a fully interactive TUI workout editor in the first pass.
- Supporting every possible Garmin workout edge case on day one.
- Guessing undocumented target semantics that we cannot validate.

## Product Direction
### 1) Add a dedicated top-level `workouts` command
- Match the existing top-level structure (`activities`, `health`, `devices`, `profile`, `sync`).
- Keep workout lifecycle separate from completed activities.

### 2) Make file-first authoring the MVP
- Nested step trees do not map cleanly to one-shot flags.
- The CLI should generate, validate, inspect, clone, and submit workout draft files.
- The initial draft format should be JSON, not YAML:
  - matches the API payload directly
  - matches existing `serde_json` usage
  - avoids adding a new parsing layer unless it clearly improves UX

### 3) Treat type-discovery as part of the public workflow
- Users need to know what `stepType`, `targetType`, `conditionType`, sport type, stroke type, and equipment type values are valid.
- The CLI should expose commands to fetch and print these definitions.

### 4) Defer flag-only step composition unless it proves ergonomically sound
- A command surface like `--step-type`, `--target-type`, `--repeat`, `--parent-repeat`, `--zone`, `--target-value-one` will get brittle quickly.
- If we later want shell-native authoring, do it as a second layer on top of a validated draft model.

## Proposed CLI Surface
### Phase 1: Read + basic lifecycle
- `garmin workouts list`
- `garmin workouts get <id>`
- `garmin workouts download <id> --output draft.json`
- `garmin workouts delete <id>`
- `garmin workouts schedule <workout-id> --date YYYY-MM-DD`
- `garmin workouts unschedule <schedule-id>`
- `garmin workouts types`
- `garmin workouts sport-types`

### Phase 2: Draft authoring workflow
- `garmin workouts init --sport running --name "Easy Intervals" --output workout.json`
- `garmin workouts clone <id> --output workout.json`
- `garmin workouts validate --file workout.json`
- `garmin workouts create --file workout.json`
- `garmin workouts update <id> --file workout.json`

### Phase 3: Guided construction helpers
Preferred helpers:
- `garmin workouts draft add-step --file workout.json ...`
- `garmin workouts draft add-repeat --file workout.json ...`
- `garmin workouts draft show --file workout.json`

Important constraint:
- Do not attempt a large mutating DSL until the underlying workout draft model and validation rules are stable.

## Draft Format Strategy
### Recommended draft format
- Introduce a local `WorkoutDraft` model that corresponds to OpenAPI `WorkoutCreate`.
- This draft should omit server-owned fields by default:
  - `workoutId`
  - `ownerId`
  - `createdDate`
  - `updatedDate`
  - `stepId`

### Builder behavior
- `init` creates a minimal valid scaffold with one segment and no steps.
- `clone` fetches an existing workout and strips server-owned IDs and timestamps.
- `validate` checks:
  - required workout fields
  - valid segment ordering
  - valid step ordering
  - repeat-group structure
  - allowed target fields per target type
  - allowed end-condition fields per condition type
- `create` posts the validated draft.
- `update` sends a full workout payload for the target id.

### Why this is the right MVP
- It supports real construction without inventing a brittle nested-flag grammar.
- It keeps manual editing available when needed.
- It lets us add richer builder helpers incrementally.

## Example Authoring Flow
### Example workout
- 10 min running at Z2
- 3 x:
  - 4 min at Z4
  - 3 min at Z3
- 10 min running at Z1

### Intended user workflow
Assumption:
- `Z1` to `Z4` mean heart-rate zones, so the target type is `heart.rate.zone`.

Preferred future CLI flow:
```bash
garmin workouts init \
  --sport running \
  --name "10' Z2 / 3x(4' Z4, 3' Z3) / 10' Z1" \
  --output workout.json

garmin workouts draft add-step \
  --file workout.json \
  --step-type warmup \
  --end-condition time \
  --seconds 600 \
  --target-type heart.rate.zone \
  --zone 2

garmin workouts draft add-repeat \
  --file workout.json \
  --iterations 3 \
  --steps-file repeat_block.json

garmin workouts draft add-step \
  --file repeat_block.json \
  --step-type interval \
  --end-condition time \
  --seconds 240 \
  --target-type heart.rate.zone \
  --zone 4

garmin workouts draft add-step \
  --file repeat_block.json \
  --step-type recovery \
  --end-condition time \
  --seconds 180 \
  --target-type heart.rate.zone \
  --zone 3

garmin workouts draft add-step \
  --file workout.json \
  --step-type cooldown \
  --end-condition time \
  --seconds 600 \
  --target-type heart.rate.zone \
  --zone 1

garmin workouts validate --file workout.json
garmin workouts create --file workout.json
```

### MVP-compatible fallback workflow
Until guided draft-editing helpers exist, the MVP flow should be:
1. Run `garmin workouts init --sport running --name ... --output workout.json`.
2. Edit `workout.json` manually to add:
   - one 600-second executable step at HR zone 2
   - one repeat group with `numberOfIterations = 3`
   - inside that repeat group:
     - one 240-second executable step at HR zone 4
     - one 180-second executable step at HR zone 3
   - one 600-second executable step at HR zone 1
3. Run `garmin workouts validate --file workout.json`.
4. Run `garmin workouts create --file workout.json`.

### Validation expectations for this example
- Every work interval/recovery block should use `targetType = heart.rate.zone`.
- The target zone should be expressed via `zoneNumber`, not `targetValueOne`/`targetValueTwo`.
- The middle block should serialize as a `RepeatGroupDTO` with:
  - `numberOfIterations = 3`
  - `endCondition = iterations`
  - nested executable steps for the 4-minute and 3-minute blocks

## Endpoint Matrix
### Confirmed on `connectapi` and compatible with current auth flow
- `GET /workout-service/workouts`
- `POST /workout-service/workout`
- `GET /workout-service/workout/{workoutId}`
- `GET /workout-service/workout/FIT/{workoutId}`
- `POST /workout-service/schedule/{workoutId}`
- `GET /workout-service/workout/types`
- `GET /workout-service/workout/types/sport`

### Present in local spec but still need live confirmation on `connectapi`
- `PUT /workout-service/workout/{workoutId}`
- `DELETE /workout-service/workout/{workoutId}`
- `DELETE /workout-service/schedule/{scheduleId}`

### Design implication
- Build the Rust surface for all CRUD endpoints.
- Treat `PUT` and `DELETE` support as implementation work plus live-smoke validation, not as already-proven assumptions.

## Required Code Changes
### 1) Transport layer
- Extend `GarminClient` with:
  - `put_json`
  - `delete`
  - optionally `delete_json` if Garmin returns a body for some delete endpoints
- Reuse existing bearer header construction and status handling.

### 2) Models
- Add workout models in `crates/garmin-cli/src/models/`:
  - `WorkoutSummary`
  - `Workout`
  - `WorkoutDraft`
  - `WorkoutSegment`
  - `WorkoutStep`
  - `ExecutableStep`
  - `RepeatGroup`
  - type-definition models for workout enums
- Keep serde mappings permissive where the reverse-engineered API is still evolving.

### 3) CLI surface
- Add `Workouts` subcommands in `crates/garmin-cli/src/main.rs`.
- Add new command handlers under `crates/garmin-cli/src/cli/commands/workouts.rs`.
- Re-export and wire command dispatch consistently with existing modules.

### 4) Draft tooling
- Add logic to:
  - create minimal draft skeletons
  - strip server-owned fields from cloned workouts
  - validate draft structure before submission
- Validation should be local and deterministic before any network call.

### 5) Output behavior
- Read commands should pretty-print JSON unless a more ergonomic summary is clearly better.
- `list` should default to a concise table, matching `activities list`.
- Mutating commands should print the resulting id or schedule id when available.

## Execution Plan
### Phase 1: Type Discovery and Transport Foundation
Goal: remove speculation from the API surface before building UX.

#### Tasks
- Add `put_json` and `delete` helpers to `GarminClient`.
- Add workout type-discovery commands:
  - `workouts types`
  - `workouts sport-types`
- Capture live fixtures for:
  - `/workout-service/workout/types`
  - `/workout-service/workout/types/sport`
- Validate target type mappings from live responses, especially:
  - `pace.zone`
  - `swim.css.offset`
  - `swim.instruction`
  - any `open`-like target key if present

#### Acceptance Criteria
- We have fixture-backed evidence for the actual target types returned by Garmin.
- Transport layer can perform authenticated `PUT` and `DELETE`.

### Phase 2: Read Surface
Goal: support safe inspection before mutation.

#### Tasks
- Implement `workouts list`.
- Implement `workouts get`.
- Implement `workouts download`.
- Add serde models or fallback JSON output for workout payloads.
- Add tests and fixtures for list/detail/download flows.

#### Acceptance Criteria
- Users can list workouts, fetch a workout, and download or clone a source workout payload.

### Phase 3: Draft Lifecycle
Goal: enable practical authoring without forcing users to handcraft raw payloads from scratch.

#### Tasks
- Implement `workouts init`.
- Implement `workouts clone`.
- Implement `workouts validate`.
- Introduce `WorkoutDraft` serialization format.
- Strip server-owned fields from cloned payloads.
- Add validation for:
  - segment ordering
  - step ordering
  - repeat structure
  - target-type-specific fields

#### Acceptance Criteria
- A user can generate a draft, edit it, validate it locally, and understand validation failures before sending anything to Garmin.

### Phase 4: Create + Schedule
Goal: complete the most important write path.

#### Tasks
- Implement `workouts create --file`.
- Implement `workouts schedule`.
- Return created workout id and scheduling response clearly.
- Add tests for create request shaping and schedule payload shaping.

#### Acceptance Criteria
- A validated draft can be created and then scheduled from the CLI.

### Phase 5: Update + Delete + Unschedule
Goal: complete CRUD and scheduling cleanup.

#### Tasks
- Implement `workouts update <id> --file`.
- Implement `workouts delete <id>`.
- Implement `workouts unschedule <schedule-id>`.
- Add live smoke tests or manual verification notes for each mutating endpoint.

#### Acceptance Criteria
- Full CRUD and schedule/unschedule are implemented behind the same `workouts` command area.
- Any endpoint that proves unsupported on `connectapi` is documented immediately and removed from help text.

### Phase 6: Guided Draft Editing Helpers
Goal: reduce manual JSON editing after the core lifecycle is stable.

#### Tasks
- Design `workouts draft add-step`.
- Design `workouts draft add-repeat`.
- Decide how nested repeat targeting should be addressed:
  - by index path
  - by step id within the draft
  - by JSON pointer-like selector
- Only implement after Phase 3 validation rules are stable.

#### Acceptance Criteria
- We can add common steps and repeats without forcing raw JSON edits for every change.

## Validation Rules for Steps
### Base validation
- `stepOrder` must be present and deterministic.
- `RepeatGroupDTO.numberOfIterations` must be >= 1.
- `RepeatGroupDTO.endConditionValue` must match `numberOfIterations`.
- Top-level steps and nested steps must serialize in the structure Garmin accepts.

### Target-specific validation
- `no.target`
  - reject target values unless Garmin proves it accepts them
- `heart.rate.zone`
  - require `zoneNumber`
- `power.zone`, `cadence.zone`, `speed.zone`, `pace.zone`
  - require `targetValueOne` and `targetValueTwo` unless Garmin proves single-value variants exist
- `swim.css.offset`, `swim.instruction`
  - mark as supported only after live type and payload examples are captured

### Condition-specific validation
- `time` -> `endConditionValue` is seconds
- `distance` -> `endConditionValue` is meters
- `calories` -> `endConditionValue` is calories
- `iterations` -> repeat-group only
- `lap.button`
  - likely no numeric value, but confirm against live examples before making it a first-class builder option

## Testing Plan
### Unit tests
- `GarminClient` `put_json` and `delete` status handling.
- Draft validation rules.
- Clone stripping rules for server-owned fields.
- CLI argument parsing for new `workouts` subcommands.

### Fixture-backed integration tests
- workout list response
- workout detail response
- workout types response
- sport types response
- create request body shaping
- schedule request body shaping
- update request body shaping

### Manual or authenticated smoke tests
- `workouts create`
- `workouts update`
- `workouts delete`
- `workouts schedule`
- `workouts unschedule`

## Risks & Tradeoffs
- The biggest risk is assuming enum mappings from stale reverse-engineered code instead of live type-discovery responses.
- `PUT` and `DELETE` may be present in the spec but not actually supported on `connectapi` for every account or region.
- A flag-only builder will become hard to understand as soon as nested repeats and multiple target types are involved.
- A JSON draft workflow is less magical, but it is safer and easier to validate.

## Open Questions
- Does `PUT /workout-service/workout/{workoutId}` actually work on `connectapi` with bearer auth for our account?
- Does `DELETE /workout-service/workout/{workoutId}` actually work on `connectapi` with bearer auth for our account?
- Does `DELETE /workout-service/schedule/{scheduleId}` work on `connectapi`, or is unschedule web-proxy only?
- What exact target type list does Garmin return from `/workout-service/workout/types` on a live account?
- Is there a live target type equivalent to `open`, or is the Python helper enum stale or incorrect?
- Do we want JSON draft files only, or JSON first with YAML as a later ergonomic layer?

## Recommended Implementation Order
1. Add `put_json` and `delete` to the API client.
2. Implement `workouts types` and capture live fixtures.
3. Implement `workouts list`, `get`, and `download`.
4. Implement draft models and `workouts init`, `clone`, `validate`.
5. Implement `workouts create` and `schedule`.
6. Implement `workouts update`, `delete`, and `unschedule`.
7. Add guided draft-editing helpers only after the draft model is stable.
8. Update README and help text only after live endpoint confirmation for each write path.

## Success Criteria
- `garmin workouts` becomes a coherent top-level feature area.
- Users can create a valid workout draft without reverse-engineering raw payloads by hand.
- Target types are sourced from live discovery responses rather than guesswork.
- CRUD support is implemented only where the underlying `connectapi` endpoints are verified.
- Help text and docs reflect only confirmed behavior.
