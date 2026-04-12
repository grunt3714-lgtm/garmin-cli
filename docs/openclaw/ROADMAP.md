# OpenClaw Health Coach Backend Roadmap

This fork is being shaped to support an OpenClaw skill that can help with health, recovery, endurance training, and workout planning.

## Priorities

### 1. Backfill-accurate sync
Goal: make historical local data trustworthy enough for coaching and trend analysis.

Why this matters:
- recovery and training advice depends on trends, not only current values
- historical HRV, sleep, VO2 max, race predictions, and training status must reflect the requested date
- stale "latest" values copied into past rows undermine analysis quality

Target outcomes:
- date-accurate historical daily health rows
- date-accurate performance metrics
- safe backfill/resync for date ranges
- reliable offline analysis for OpenClaw skill summaries

### 2. Workout CRUD and builder
Goal: let OpenClaw not only analyze health and training, but also generate and schedule workouts.

Why this matters:
- the next step after analysis is action
- OpenClaw should be able to propose, create, and schedule structured workouts
- this enables training plans, recovery sessions, interval workouts, and race-specific preparation

Target outcomes:
- `garmin workouts` command area
- workout draft init/validate/create flows
- schedule/unschedule support
- foundation for OpenClaw-generated workouts from natural language

### 3. Sync CLI cleanup
Goal: make sync easier to understand and more trustworthy for users.

Why this matters:
- clearer sync flows reduce confusion
- simpler CLI surface improves maintainability
- strong docs reduce drift between README and actual behavior

Target outcomes:
- more obvious sync commands and summaries
- docs that match real CLI behavior
- reduced dead surface area

## Recommended execution order
1. Backfill-accurate sync
2. Workout CRUD and builder
3. Sync CLI cleanup

## Success criteria for OpenClaw integration
- OpenClaw can answer trend-based health and training questions from accurate historical data
- OpenClaw can connect recovery state to training load and recent workouts
- OpenClaw can create and schedule workouts based on the user's goals and readiness
- the CLI remains stable and understandable enough to serve as a long-term assistant backend

## Current validation notes (2026-04-12)

### Confirmed working live
Direct CLI queries are already returning historical/date-scoped values for several coaching-relevant metrics:
- `garmin health sleep --days 7`
- `garmin health training-readiness --days 14`
- `garmin health training-status --days 14`
- `garmin health summary`

This means the direct health query surface is already useful for OpenClaw coaching.

### Remaining concern for local/offline coaching
The source still shows likely historical-sync correctness issues in some areas:
- `racepredictions/latest` still appears in source
- `mostRecentTrainingStatus` still appears in source
- sync code still contains logic that may write latest-style values into dated rows

### Practical interpretation
- Direct CLI health queries are already strong enough for interactive coaching.
- Sync accuracy is still the highest-value engineering target for trustworthy historical offline analysis.
- Workout CRUD/builder remains the strongest next capability after sync accuracy.
