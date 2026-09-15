# Workout Tracker App — Technical Reference

Companion to `Workout_App_Project_Brief.md`. This file exists so any future work session — Claude or otherwise — can pick up the codebase without re-deriving the architecture.

---

## 1. Architecture

- **Single self-contained HTML file** (`Workout_Tracker.html`) — inline `<style>` and `<script>`, no build step, no external dependencies
- Rendered as a Claude artifact — opens directly in the Claude web/mobile app
- **Vanilla JavaScript**, no framework — all rendering is manual DOM string templating (`innerHTML` swaps per view), not React/Vue
- Client-side only — no server code, no database Zach controls directly

---

## 2. Persistence — `window.storage`

Simple key-value API provided by the Claude artifacts runtime. Not a traditional relational database.

| Key | Contents | Scope |
|---|---|---|
| `exercise-library` | JSON array of exercise definitions | Personal (`shared: false`, default) |
| `workout-sessions` | JSON array of all finished sessions, newest first | Personal |
| `workout-templates` | JSON array of named exercise presets (see §3) | Personal |
| `active-session` | The in-progress session, if any (survives app close/reopen) | Personal |
| `profile` | `{ name: string }` | Personal |

**Limits:** 5MB per key, keys under 200 chars, text/JSON only (no binary/image data). All calls wrapped in try/catch; failed saves now surface a visible toast warning (`⚠ Save failed — see Settings`) rather than failing silently.

**Known constraint:** storage is scoped per viewing user by the platform — this is documented behavior but has not been independently verified with a second real account. Before trusting it with someone else's data (Phase 2), test with one real second user first.

---

## 3. Data Schemas

### Exercise (library entry)
```json
{
  "id": "string (uid)",
  "name": "string",
  "day": "Push | Pull | Legs | Cardio | \"\" (Any)",
  "muscleGroup": "Chest | Back | Shoulders | Arms | Legs | Core | Cardio | Other",
  "usesWeight": true,
  "sets": 4,
  "reps": 10,
  "weight": 0,
  "restSets": 75,
  "restNext": 60
}
```
`day` (training-split tag) and `muscleGroup` (anatomical tag) are independent fields — kept separate rather than merged, since they answer different questions ("what split is this for" vs. "what does it work"). Every list that shows the exercise library (Exercises tab, the in-workout "Add exercise" dropdown, the template builder) groups by `muscleGroup` (alphabetical, "Other" last) and sorts alphabetically within each group via `groupLibraryForDisplay()`. Exercises without a valid `muscleGroup` (older entries created before this field existed, or AI Trainer imports, which don't set it) fall into "Other" until edited.

### Session (workout-sessions entry)
```json
{
  "id": "string (uid)",
  "date": "ISO datetime — session start",
  "finishedAt": "ISO datetime — session end (set on finish)",
  "exercises": [
    {
      "libId": "string — links back to exercise-library id",
      "name": "string",
      "usesWeight": true,
      "targetReps": 10,
      "targetWeight": 0,
      "restSets": 75,
      "restNext": 60,
      "sets": [
        { "reps": 10, "weight": 135, "done": true }
      ]
    }
  ]
}
```

### Derived stats (`sessionStats()` helper, computed on the fly — not stored)
- `totalWeight` — sum of `reps * weight` across all `done` sets where `usesWeight` is true
- `totalSets` — count of `done` sets
- `durationMin` — `(finishedAt - date)` in minutes, null if session was never finished

### Template (workout-templates entry)
```json
{
  "id": "string (uid)",
  "name": "string — e.g. \"Push Day A\"",
  "exerciseIds": ["string — library id", "..."]
}
```
Templates store only library IDs, not a frozen copy of sets/reps/weight. If a referenced exercise is later deleted from the library, it's silently skipped when the template is started.

**Weight/reps preload:** `buildSessionExercise()` (used by both template-start and manual add) calls `getLastPerformance(libId, name)`, which scans `sessions` (newest-first) for the most recent finished session containing that exercise and returns its `done` sets in order. Each new set slot preloads that set-index's actual last-performed reps/weight; only sets beyond what was previously logged (or an exercise never done before) fall back to the library's static `reps`/`weight` defaults. Rest times (`restSets`/`restNext`) always come from the library directly — they're exercise-level, not session-level, so they never went stale.

### Set editing during a workout
Every not-yet-`done` set's reps/weight is editable, not just the next one to log — lets you plan a pyramid (e.g. 10→15→20 lb) before starting. Logging itself is still sequential (the OK button stays disabled until a set is "next" — all prior sets done). Editing one set's value auto-fills forward: later not-done sets that still match the *old* value update to the new one, contiguously, stopping at the first later set that's already been changed to something different (that set — and everything after it — is left alone, since it's considered manually set).

### Personal records
No separate storage — computed on demand (`bestValueForExercise()`, `allPRs()`) by scanning `done` sets across `sessions` + the active session. Definition: for weighted exercises, PR = highest single-set `weight` ever logged; for bodyweight exercises, PR = highest single-set `reps`. A set that sets a new PR gets `isPR: true` written onto it at log time (shown as 🏆 in the Workout view, History, and the Progress tab's Personal Records card).

---

## 4. AI Personal Trainer Integration

Calls the Anthropic Messages API directly from the client (per the documented "AI in artifacts" capability — no API key handled client-side, platform manages it).

- **Endpoint:** `https://api.anthropic.com/v1/messages`
- **Model:** `claude-sonnet-4-6`
- **max_tokens:** `1000` (kept intentionally low — prompt asks for compact/minified JSON, 12-16 exercises, to stay in budget)
- **Prompt shape:** equipment list + goal + days/week → instructed to return ONLY minified JSON: `{"exercises":[{"name","day","usesWeight","sets","reps","restSets","restNext"}]}`
- **Response handling:** strips any stray markdown fences, `JSON.parse`s the result, renders a checkbox preview list, only adds user-selected + non-duplicate exercises to the library on confirm
- **Failure handling:** wrapped in try/catch — shows a plain-language error box (not a silent failure) if the request fails or JSON parsing fails

---

## 4a. Plate Calculator

Pure function, no storage. `calcPlates(target, bar)` greedily fills one side using standard plate sizes `[45, 35, 25, 10, 5, 2.5]` lb, returns `{ used: number[], remainder: number }` — `remainder` flags weight per side that can't be made exactly with those sizes (e.g. odd/micro-plate targets). Opened from a 🏋 button on each weighted exercise during an active workout (pre-filled with that exercise's target weight) or standalone from Settings (starts at 0). Bar weight defaults to 45 lb and is remembered across opens for the session (`lastBarWeight`, in-memory only, not persisted).

---

## 5. Design Tokens

| Token | Value | Use |
|---|---|---|
| `--bg` | `#0B0E14` | App background |
| `--surface` | `#151A24` | Cards |
| `--surface-2` | `#1D2330` | Inputs, nested surfaces |
| `--border` | `#2A3140` | Dividers, outlines |
| `--text` | `#ECEEF2` | Primary text |
| `--text-muted` | `#8A93A6` | Secondary text |
| `--accent` | `#F2A93B` | Primary actions, rest timer ring, AI feature |
| `--good` | `#4ADE80` | Completed sets, success states |
| `--danger` | `#E8604A` | Destructive actions, errors |
| Font | System stack (`-apple-system, Segoe UI, Roboto…`) | No external font loading (offline-safe) |

Kept intentionally distinct from Strong's visual language (see Project Brief §4, legal notes).

---

## 4b. In-session exercise controls

Each exercise card in an active workout (not the library) carries its own action row: **▲/▼** reorder (`moveExercise(exIdx, dir)` — splices the exercise to its new index in `active.exercises`, disabled at the ends), **✎** edit (`openSessionExerciseEditModal(exIdx)` — a small modal for that exercise's `restSets`/`restNext`, scoped to this session only, not the library default), **🏋** plate calculator (weighted exercises only), and **✕** remove exercise.

An earlier version gated a fresh exercise behind a full-card "Start Exercise" step before showing any sets — reverted (Sep 2026) in favor of showing sets immediately (as originally built) with rest-time editing moved to the ✎ icon instead, since the gate blocked the actual workout for something that's really just an occasional adjustment.

Each not-yet-`done` set also gets its own **✕** (`removeSetFromExercise`, blocked with a toast if it's the exercise's last remaining set — remove the whole exercise instead), and every exercise has a **+ Add Set** button (`addSetToExercise`) that appends a new set copying the last set's reps/weight. Both operate on `active.exercises[exIdx].sets` directly and call `saveActive()`.

### Warm-up set (**W** toggle)
`toggleWarmupSet(exIdx)` adds or removes a single `{ reps, weight, done: false, isWarmup: true }` set at the *front* of `ex.sets` (default weight: 50% of the exercise's target weight, rounded to the nearest 5 lb; reps: the target reps — both editable like any set). Toggling off is blocked if that warm-up set is already logged (remove it manually via its own ✕ instead). The **W** icon shows active (`wt-iconbtn-active`, accent-filled) whenever a warm-up set currently exists on that exercise.

A warm-up set renders with a blue outline (`--info` token, `.wt-set-warmup`) and shows "W" instead of a set number; the numbering shown to the working sets skips over it so they still read 1, 2, 3. It gates the sequence like any other set (must be logged before set 1 is "next") but is deliberately excluded everywhere real performance is measured: `sessionStats()`, `bestValueForExercise()` (both as a candidate PR and as the excluded "this set" when checking one), `allPRs()`, and `getLastPerformance()` (so it never gets preloaded as next time's working weight). History still shows it, prefixed "W:", rather than hiding it.

## 4c. Starter library defaults, and normalizing an existing one

`DEFAULT_LIBRARY`'s 15 seed exercises all use `sets: 3, reps: 10` (normalized Sep 2026 — previously varied per exercise, e.g. Leg Extension was 3×13, Chest Press 4×8). **This only affects a brand-new, never-initialized `exercise-library`** — `loadAll()` only falls back to `DEFAULT_LIBRARY` when `window.storage.get("exercise-library")` returns nothing, so anyone with an already-populated library (essentially everyone past first launch) keeps their existing per-exercise sets/reps regardless of this constant. Settings → "Normalize to 3×10" (`wt-normalize-btn`) is the retroactive fix: loops the live `library` array, sets every entry's `sets = 3, reps = 10` (leaves weight/day/muscleGroup/rest times untouched), saves, re-renders. It only touches the library, not exercises already added to today's `active` session — those were instantiated from the library at add-time and keep whatever sets/reps they were given then (editable inline same as any set, or remove and re-add after normalizing to pick up the change).

## 4d. Elapsed session timer (top of screen)

`#wt-elapsed-timer` lives in the static topbar markup (not inside `#wt-view`), so it survives every `render()`/tab switch untouched — `startElapsedTimer()` ticks it every second via `setInterval`, computed from real wall-clock time (`Date.now() - new Date(active.date)`), not an incrementing counter, so a throttled/backgrounded tab just shows a jump forward when it resumes rather than drifting wrong. Started on `startWorkout()`, `startWorkoutFromTemplate()`, and on app init if an `active` session was already in progress (reopening mid-workout); stopped and blanked by `stopElapsedTimer()` in `finishWorkout()`.

## 4e. Cardio mode, and body weight for bodyweight exercises

**Cardio exercises** (Sep 2026): an exercise's `mode` is `"strength"` (default — everything before this point in the doc) or `"cardio"`. A cardio exercise skips `usesWeight`/`reps`/`weight` entirely and instead has `durationMin`/`distanceMi` as its per-exercise targets; its sets are `{ durationMin, distanceMi, done }` instead of `{ reps, weight, done }`. The New Exercise modal has a **Type** selector that shows/hides the two field groups live (`wt-f-usesweight-row`/`wt-f-reps-wrap`/`wt-f-weight-wrap` vs `wt-f-duration-wrap`/`wt-f-distance-wrap`, toggled via `.wt-hidden`). `buildSessionExercise()` and `addSetToExercise()` both branch on `lib.mode`/`ex.mode`. The generic per-set input handler (auto-fill-forward included) needed no changes — it already keys off `data-field`, so `durationMin`/`distanceMi` just work as field names like `reps`/`weight` do.

Cardio is intentionally minimal for now: no PR tracking (`bestValueForExercise`/`allPRs`/`logSet`'s PR check all skip `mode === "cardio"` outright — there's no clean single-number "PR" for a walk), no progression chart, and the 🏋 plate calculator / **W** warm-up toggle don't render on a cardio exercise's action row since neither applies. `sessionStats()` accumulates cardio into separate `cardioMin`/`cardioMi` totals, kept apart from `totalWeight`/`totalSets` — shown in History's summary line and the Progress tab's All-Time card (mi only, when > 0), and in each cardio exercise's per-set History line as "Xmin / Ymi".

**Body weight** (`profile.bodyweight`, Settings): a plain number, not tracked over time, used only so `sessionStats()` can estimate weight moved on bodyweight (`usesWeight: false`, non-cardio) exercises as `reps × bodyweight` — the same total-weight-moved metric weighted exercises already had. Saving either the display name or body weight in Settings now merges into the existing `profile` object (`{ ...(profile||{}), ... }`) rather than replacing it, so setting one no longer wipes out the other.

## 5a. Rest Timer (inline, manual-start)

Replaced the original full-screen overlay timer (Sep 2026) after feedback that it was hard to read and blocked the rest of the workout. Now: logging a set that triggers rest sets `restState = { exIdx, afterSetIdx, kind, totalSeconds, remaining, running: false, nextLabel }` and renders an inline card directly under that set row (inside the same exercise's card), not a fixed overlay.

- **Idle state** (`running: false`): shows the configured rest duration in an editable `<input>`, plus Start/Skip. Nothing counts down until Start is tapped — this is intentional (manual start), not a bug: it also means the duration is genuinely editable before it begins, not just adjustable in ±15s increments after the fact.
- **Running state**: countdown + Next label + "+15s"/Skip. The interval (`restInterval`) updates only the `#wt-inline-timer-num` text node directly each tick (not a full `render()`) — same approach the old overlay used, kept for the same reason: a full re-render every 250ms while resting would disrupt any other set inputs the user has focus on elsewhere on the page.
- Skip (idle or running) or hitting 0 both call `finishRest()`, which clears `restState` and re-renders — this is what unlocks the next set's OK button (via `isNext` in `renderWorkout()`), same gating as before.
- Logging order is still sequential regardless of rest state — but note logging the *next* set early (before Skip/finish) is now possible without any special handling, since `logSet()` unconditionally overwrites `restState` at the top. Un-gating the rest timer from set-logging was a deliberate side effect of going inline, not an accident.
- If `restSets`/`restNext` is 0 for an exercise, no rest state is created at all (unchanged from prior behavior).

---

## 6. Known Limitations (structural, not bugs)

- No push notifications — pure web page, can't alert outside an open session
- No native device integrations (Garmin, Apple Health, etc.) — would require a real backend with OAuth
- No image storage — 5MB text-only cap rules out progress photos in current form
- No true multi-device real-time sync beyond whatever the artifact storage platform provides
- File download for backups intentionally avoided (sandboxed iframe reliability unknown) — backup/restore is copy-paste text instead

---

## 7. File Locations (as of last build)

- App: `/mnt/user-data/outputs/Workout_Tracker.html`
- Companion spreadsheet: `/mnt/user-data/outputs/Recomp_Tracker.xlsx` (Dashboard, Program, Weight Log, Workout Log, Nutrition Log, Supplement & Habits Log, Stretch Routine tabs)
