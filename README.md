# SETLOG

A personal workout logging app built around one gym, one set of equipment, and real training goals — not a generic template. Started as a Claude artifact; this repo lets it also run standalone (e.g. GitHub Pages).

## Run it

Just open `Workout_Tracker.html` in a browser — no build step, no dependencies. Data is saved to that browser's `localStorage` when run outside Claude (see `Workout_App_Technical_Reference.md` for how the storage shim works).

## Features

- Exercise library grouped by muscle group, with day tags (Push/Pull/Legs/Cardio)
- Live workout logging: sets, reps, weight (editable ahead of time, auto-fills forward), warm-up sets, plate calculator
- Inline rest timer (editable, manual start), elapsed session timer
- Workout templates, personal record tracking, reorderable exercises
- Cardio mode (duration/distance) alongside strength exercises
- Progress charts, full history, text-based backup/restore
- Optional AI-generated routines (needs your own Anthropic API key when run outside Claude — see Settings)

## Docs

- [`Workout_App_Project_Brief.md`](Workout_App_Project_Brief.md) — vision, roadmap, decision log
- [`Workout_App_Technical_Reference.md`](Workout_App_Technical_Reference.md) — architecture, data schemas, feature internals

Personal project — not affiliated with or a replacement recommendation against any commercial fitness app.
