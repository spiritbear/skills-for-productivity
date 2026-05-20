---
name: tonal-workout
description: Pull Tonal workout data and update daily notes with detailed workout summaries including exercises, sets, reps, and volume
---

You are updating the user's Obsidian daily notes with their Tonal workout data. Follow this process carefully.

## Prerequisites

- **toneget** is installed at `~/.local/share/toneget/`
- 1Password CLI (`op`) must be signed in — credentials are resolved via `op run` from `op://Private/Tonal`
- The fetch script is at `~/.local/share/toneget/fetch_recent.py`

## Resilience

Each step has a happy path and known failure modes. When a step fails, capture the exit code and stderr, match against the failure modes below, apply the fix, and retry the step. **Cap: 5 attempts per step.** If you hit 5 without success, stop the skill and report:

- The step that failed
- For each attempt: the command, the stderr, and the fix you tried
- What you'd recommend the user do next

**Recoverable failures — diagnose and retry (or adapt and continue):**
- Schema variants in JSON keys (exercise / weight / reps / volume) — adapt per the variant list in Step 3; do not retry
- No workouts in the requested window — not a failure; report and stop
- `## Workout` section missing in a target daily note — insert it per Step 5
- Transient network or rate-limit error from the fetch script — wait 5 seconds, retry once

**Always escalate — never auto-fix and never retry:**
- `op` not signed in → ask the user to run `op signin`, stop
- `op://Private/Tonal` item missing or unreadable → ask the user to set it up, stop
- Tonal API auth failure (bad email/password in 1Password) → ask the user to update the 1Password item, stop
- Daily note for a given date does not exist → skip that date, do not create the file
- Daily note already has populated workout data — ask before overwriting
- Persistent network failures after one retry
- JSON structure doesn't match expected fields even after adapting — print the first workout's keys and stop

## Step 1: Parse Arguments

The user may provide arguments to control scope:

```
/tonal-workout              # fetch last 7 days of workouts
/tonal-workout 30           # fetch last 30 days
/tonal-workout 2026-03-23   # fetch a specific date
```

- If the argument looks like a date (YYYY-MM-DD), use `--date <date>`
- If the argument is a number, use `--days <number>`
- If no argument, default to `--days 7`

## Step 2: Fetch Workout Data

Run the fetch script via 1Password CLI to inject credentials:

```bash
op run --env-file ~/.local/share/toneget/.env -- python3 ~/.local/share/toneget/fetch_recent.py <args> 2>/dev/null
```

This uses `op run` to resolve `TONAL_EMAIL` and `TONAL_PASSWORD` from the user's 1Password Private vault, then outputs a JSON array of workouts to stdout. Capture it.

If the command fails, check:
1. Is the user signed into `op`? They may need to run `op signin` first.
2. Is the network available?
3. Are credentials valid? The Tonal item is in the Private vault (`op://Private/Tonal`).

## Step 3: Parse Workout JSON

Each workout object contains:

| Field | Description |
|-------|-------------|
| `beginTime` | ISO timestamp (e.g., `2026-03-23T14:30:00Z`) — use the **date portion** to match daily notes |
| `endTime` | ISO timestamp |
| `workoutType` | `PROGRAM`, `ON_DEMAND`, `QUICK_FIT`, `LIVE`, `MOVEMENT`, `ASSESSMENT`, or `Custom` |
| `customWorkoutName` | Name of custom workout (only present for custom workouts) |
| `workoutName` | Name of the workout program/class |
| `instructorName` | Tonal coach name |
| `totalVolume` | Total weight lifted in lbs |
| `totalReps` | Total reps performed |
| `totalTime` | Duration in seconds |
| `workoutSetActivity` | Array of set-level data (see below) |

Each item in `workoutSetActivity`:

| Field | Description |
|-------|-------------|
| `exerciseName` | Name of the exercise |
| `setOrder` | Which set number |
| `actualReps` | Reps completed |
| `actualWeight` | Weight used in lbs |
| `totalVolume` | Volume for this set (weight x reps) |
| `moveType` | Type of movement |

**IMPORTANT**: The Tonal API field names may vary. Inspect the actual JSON keys when parsing. Common variations:
- Exercise name: `exerciseName`, `exercise_name`, `name`, or nested under an `exercise` object
- Weight: `actualWeight`, `weight`, `resistance`
- Reps: `actualReps`, `reps_completed`, `completedReps`
- Volume: `totalVolume`, `volume`

Adapt to whatever fields are present in the actual response.

## Step 4: Group Workouts by Date

Extract the date from `beginTime` for each workout. Convert to `YYYY-MM-DD` format. Group workouts by date — a user may have multiple workouts on the same day.

## Step 5: Update Daily Notes

For each date that has workouts:

1. **Find the daily note** at `07-Daily/YYYY/MM/YYYY-MM-DD.md`
   - If it doesn't exist, do NOT create it — skip that date and inform the user

2. **Build the workout section content**. Format as follows:

For the summary table (one row per workout that day):

```markdown
## Workout

| Workout | Type | Duration | Volume (lbs) | Reps |
| ------- | ---- | -------- | ------------ | ---- |
| Upper Body Blast | On-Demand | 35 min | 4,250 | 120 |
```

- **Workout**: Use `workoutName`, `customWorkoutName`, or the workout type as fallback
- **Type**: Human-readable version of `workoutType` (e.g., `ON_DEMAND` -> `On-Demand`, `PROGRAM` -> `Program`)
- **Duration**: Convert `totalTime` seconds to `X min` format
- **Volume/Reps**: Format numbers with commas for readability

If an instructor is present, add a line: `**Coach**: {instructorName}`

For the exercises breakdown, aggregate sets by exercise name:

```markdown
### Exercises
| Exercise | Sets | Reps | Weight (lbs) | Volume (lbs) |
| -------- | ---- | ---- | ------------ | ------------ |
| Bench Press | 3 | 10, 10, 8 | 80, 85, 85 | 2,450 |
| Bicep Curl | 3 | 12, 12, 10 | 25, 25, 30 | 910 |
```

- **Sets**: Count of sets for that exercise
- **Reps**: Comma-separated reps per set
- **Weight**: Comma-separated weight per set (if same across all sets, just show once)
- **Volume**: Total volume across all sets for that exercise

3. **Replace the existing workout section** in the daily note. Look for the `## Workout` heading and replace everything up to the next `##` heading or the `---` separator. If there's no `## Workout` section, look for `## Workout Type` (old format) instead. If neither exists, insert the workout section before the `---` separator that precedes the navigation links.

4. **Remove the callout** — delete the `> [!note] Run...` line if present, since the data is now populated.

## Step 6: Report Results

After updating, summarize what was done:
- How many workouts were found
- Which daily notes were updated
- Any dates skipped (no daily note found)

## Error Handling

See the **Resilience** section near the top for failure modes, recovery actions, and the 5-attempt cap. The cases previously listed here (no workouts, existing populated workout data, unknown JSON structure) are covered there.
