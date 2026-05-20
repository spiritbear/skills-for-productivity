---
name: garmin-activity
description: Pull a single Garmin Connect activity by ID and append a formatted summary to the matching daily note. Use this whenever the user wants to log, capture, or fetch a Garmin activity into the vault — running, hiking, rowing, or any other activity recorded on a Garmin device. Trigger on phrases like "garmin activity", "log my run", "log my hike", "log my row", "fetch this activity", or when the user provides a Garmin activity ID (a long numeric string like `22929985702`) and asks to record it. Also trigger on "/garmin-activity <id>".
---

You are pulling a single Garmin Connect activity by ID and writing a formatted summary into the user's Obsidian daily note for the day the activity took place.

## Prerequisites

- `garmin` CLI is installed at `/opt/homebrew/bin/garmin` (the [garmin-cli](https://github.com/vicentereig/garmin-cli) Rust tool)
- The user is already authenticated. If a `garmin` call fails with an auth error, tell the user to run `garmin auth login` and stop — do not attempt to authenticate on their behalf.

## Resilience

Each step has a happy path and known failure modes. When a step fails, capture the exit code and stderr, match against the failure modes below, apply the fix, and retry the step. **Cap: 5 attempts per step.** If you hit 5 without success, stop the skill and report:

- The step that failed
- For each attempt: the command, the stderr, and the fix you tried
- What you'd recommend the user do next

**Recoverable failures — diagnose and retry (or adapt and continue):**
- JSON missing an expected metric field (older device, manual activity) — omit that row; do not retry
- Rowing field names vary (`averageStrokeCadence` / `averageStrokes` / `avgStrokeCadence`, `totalNumberOfStrokes` / `totalStrokes`) — inspect what's present in the JSON and use it
- `## Activities` section missing in the daily note — insert it per Step 7
- Transient network error from `garmin activities get` — wait 5 seconds, retry once

**Always escalate — never auto-fix and never retry:**
- `garmin` returns an auth error → tell the user to run `garmin auth login`, stop
- Activity ID returns 404 / not found → tell the user the ID may be wrong, stop
- Daily note does not exist → tell the user, stop (do not create the file)
- Persistent network failures after one retry

## Step 1: Parse the activity ID

The user invokes this as `/garmin-activity <activity-id>` or asks in natural language. The activity ID is the long numeric string Garmin Connect uses (e.g., `22929985702`). It is always required — if no ID is provided, ask the user for it.

If the user provides a Garmin Connect URL (e.g., `https://connect.garmin.com/modern/activity/22929985702`), extract the trailing numeric ID.

## Step 2: Fetch the activity

```bash
garmin activities get <activity-id>
```

This prints a JSON object to stdout. Capture it. If the command errors:
- Auth error → ask the user to run `garmin auth login` and stop
- Activity not found → tell the user the ID may be wrong and stop
- Network error → report it and stop

## Step 3: Identify the activity type

Read `activityTypeDTO.typeKey`. Map it to one of the three supported formats:

| `typeKey` value | Format to use |
|---|---|
| `running`, `trail_running`, `street_running`, `treadmill_running`, `track_running` | **Running** |
| `hiking`, `walking`, `casual_walking`, `speed_walking` | **Hiking** |
| `rowing`, `indoor_rowing`, `open_water_rowing` | **Rowing** |

If `typeKey` doesn't match any of the above, fall back to a **Generic** format (described at the bottom) and tell the user the type wasn't one of the three first-class cases.

## Step 4: Find the daily note

The activity date comes from `summaryDTO.startTimeLocal` (e.g., `2026-05-18T17:01:46.0`). Use the **date portion only** (`2026-05-18`).

Daily note path: `07-Daily/YYYY/MM/YYYY-MM-DD.md`

If the daily note doesn't exist, do **not** create it. Tell the user and stop — daily notes are user-driven and we shouldn't conjure them just to log an activity.

## Step 5: Convert units to imperial

All output is in miles and feet. The Garmin JSON is metric. Conversions:

| From | To | Formula |
|---|---|---|
| meters | miles | `m / 1609.344` |
| meters | feet | `m * 3.28084` |
| m/s | mph | `mps * 2.23694` |
| m/s | min/mile pace | `26.8224 / mps` → minutes, then split into `MM:SS` |
| seconds | duration string | `H:MM:SS` if ≥1h, else `MM:SS` |

Round distance to 2 decimals, elevation/calories to whole numbers, HR to whole numbers, pace to `MM:SS`. Format large numbers with thousands separators.

**Cadence**: `averageRunCadence` is already total steps-per-minute (both legs) for the user's Garmin device — verified empirically by cross-checking `steps / movingDuration`. Do **not** double it. Display the raw value rounded to a whole number.

## Step 6: Build the activity block

Each activity is rendered as a level-3 heading inside a top-level `## Activities` section, with a hidden HTML comment marker for idempotency.

**Marker line (always include, immediately after the `###` heading):**
```
<!-- garmin-activity-id: <activity-id> -->
```

This lets the skill detect duplicates on re-runs.

**Heading format:**
```
### {Emoji} {Activity Name} — {Type}
```

Emojis by type: 🏃 running, 🥾 hiking, 🚣 rowing, 🏋️ generic.

If `activityName` is generic-looking (e.g., starts with the location name only), still use it — but you can fall back to the type's display name if `activityName` is empty.

### Running format

```markdown
### 🏃 {activityName} — Running
<!-- garmin-activity-id: {activityId} -->

| Metric | Value |
| --- | --- |
| Distance | {miles} mi |
| Duration | {duration} (moving: {movingDuration}) |
| Avg Pace | {pace} /mi |
| Avg HR | {avgHR} bpm |
| Max HR | {maxHR} bpm |
| Elevation Gain | {gain} ft |
| Calories | {calories} |
| Cadence | {cadence} spm |
| Steps | {steps} |
| Training Effect | {trainingEffect} ({trainingEffectLabel}) |
| Location | {locationName} |

[View on Garmin Connect](https://connect.garmin.com/modern/activity/{activityId})
```

Source fields: `summaryDTO.distance`, `summaryDTO.duration`, `summaryDTO.movingDuration`, `summaryDTO.averageSpeed` (for pace), `summaryDTO.averageHR`, `summaryDTO.maxHR`, `summaryDTO.elevationGain`, `summaryDTO.calories`, `summaryDTO.averageRunCadence`, `summaryDTO.steps`, `summaryDTO.trainingEffect`, `summaryDTO.trainingEffectLabel`, `locationName`.

### Hiking format

```markdown
### 🥾 {activityName} — Hiking
<!-- garmin-activity-id: {activityId} -->

| Metric | Value |
| --- | --- |
| Distance | {miles} mi |
| Duration | {duration} (moving: {movingDuration}) |
| Avg Pace | {pace} /mi |
| Avg HR | {avgHR} bpm |
| Max HR | {maxHR} bpm |
| Elevation Gain | {gain} ft |
| Elevation Loss | {loss} ft |
| Calories | {calories} |
| Steps | {steps} |
| Training Effect | {trainingEffect} ({trainingEffectLabel}) |
| Location | {locationName} |

[View on Garmin Connect](https://connect.garmin.com/modern/activity/{activityId})
```

Same source fields as running, plus `summaryDTO.elevationLoss`. Skip cadence — not meaningful for hiking.

### Rowing format

```markdown
### 🚣 {activityName} — Rowing
<!-- garmin-activity-id: {activityId} -->

| Metric | Value |
| --- | --- |
| Distance | {miles} mi |
| Duration | {duration} (moving: {movingDuration}) |
| Avg Pace | {pace} /mi |
| Avg HR | {avgHR} bpm |
| Max HR | {maxHR} bpm |
| Avg Stroke Rate | {strokeRate} spm |
| Total Strokes | {strokes} |
| Calories | {calories} |
| Training Effect | {trainingEffect} ({trainingEffectLabel}) |

[View on Garmin Connect](https://connect.garmin.com/modern/activity/{activityId})
```

Source fields: same as running for distance/duration/HR/calories/training effect, plus `summaryDTO.averageStrokeCadence` (or `averageStrokes`, `avgStrokeCadence` — check what's present), `summaryDTO.totalNumberOfStrokes` (or `totalStrokes`). Skip elevation, steps. If `locationName` is empty (common for indoor rowing), omit the Location row.

Field names vary slightly between rowing implementations — inspect what's present in the JSON and adapt. If a field is missing, omit the row rather than printing `null`.

### Generic format (fallback)

Use this if the type isn't running/hiking/rowing. Include only the fields that are present and non-zero:

```markdown
### 🏋️ {activityName} — {Type}
<!-- garmin-activity-id: {activityId} -->

| Metric | Value |
| --- | --- |
| Distance | {miles} mi |
| Duration | {duration} |
| Avg HR | {avgHR} bpm |
| Max HR | {maxHR} bpm |
| Calories | {calories} |
| Training Effect | {trainingEffect} ({trainingEffectLabel}) |

[View on Garmin Connect](https://connect.garmin.com/modern/activity/{activityId})
```

## Step 7: Insert into the daily note

There are three cases. Inspect the daily note's content before deciding.

**Case A — `## Activities` section already exists and already contains this activity ID** (search for `<!-- garmin-activity-id: {activityId} -->`):
Replace that activity's block (from its `###` heading up to the next `###` or the next top-level `##` heading or end of section, whichever comes first). This makes re-running the skill idempotent.

**Case B — `## Activities` section exists but this ID is not in it:**
Append the new activity block to the end of the section, before the next top-level heading or the `---` separator that precedes navigation links.

**Case C — `## Activities` section does not exist:**
Insert a new `## Activities` section. Place it **immediately after the `## Workout` section** if that section exists, otherwise before the `---` separator that precedes the navigation links (`<< [[...|Yesterday]] | [[...|Tomorrow]] >>`).

The section header looks like:
```markdown
## Activities

{activity block}
```

Preserve all surrounding content. Do not touch the frontmatter, the Tasks section, the Workout section (Tonal data lives there), or the Dataview blocks at the bottom.

## Step 8: Report

Tell the user briefly:
- Activity type and name
- Daily note path that was updated
- Whether this was a new entry, a replaced duplicate, or a brand-new section

If anything was skipped or unusual (missing fields, unknown activity type, missing daily note), call it out plainly.

## Edge cases

- **Multi-sport activity** (`isMultiSportParent: true`) — these aggregate child activities. For now, treat the parent as a single block using its summary; mention to the user that it's a multi-sport parent and they may want to log child activities separately.
- **Manual activity** (`metadataDTO.manualActivity: true`) — some fields will be missing or zero. Use the generic format and only include rows that have real values.
- **Indoor rowing with no GPS** — `locationName`, `startLatitude`, `elevationGain` will be missing or null. Omit those rows.
- **Pace calculation when moving speed is 0** — guard against division by zero. If pace can't be computed, render as `—`.
- **Timezone**: always use `startTimeLocal` (not `startTimeGMT`). The user's daily note follows local calendar dates.
