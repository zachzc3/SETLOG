# Workout Tracker App — Project Brief

**Status:** Personal MVP live, in active feature development
**Working name:** SETLOG *(placeholder — rename anytime, it's cosmetic)*
**Owner:** Zach
**Live artifact:** https://claude.ai/code/artifact/e783c5ba-0627-40b9-98a4-bbc7cbcba5c9

**Confirmed personal-only (Sep 2026):** This is for Zach's own use, not a public/shared product. The Phased Approach and Legal/Naming Notes sections below were written for an eventual public-sharing path — they're kept for reference but are not active constraints. Build for "does it work for how I actually train," not for other users.

---

## 1. Vision

A workout logging app built exactly around one person's actual gym, actual equipment, and actual goals — not a generic template. Started as a personal tool to replace re-subscribing to Strong; the long-term option is open to eventually sharing with others or shipping publicly, but that's a "if it's worth it" decision, not a starting requirement.

**Phased approach:**

| Phase | Description | Trigger to move forward |
|---|---|---|
| **1 — Personal use** *(current)* | Single-user, lives inside a Claude artifact, storage tied to Zach's account | — |
| **2 — Friends pilot** | Share the artifact link with a few people, confirm per-user data separation works as expected | Phase 1 feels solid after real use |
| **3 — Real hosted app** | Proper backend (Supabase/Firebase), real auth, own domain, still just a web app (PWA) | Phase 2 shows genuine repeat usage from others |
| **4 — App Store presence** | Native/cross-platform wrapper, Apple + Google developer accounts | Phase 3 shows people want it on their home screen enough to justify $99/yr + review process |

Skipping straight to Phase 4 is the wrong move — it's the most expensive, slowest-to-iterate option and should only happen once the concept is *proven*, not while it's still being shaped.

---

## 2. Current Feature Set (built and working)

- **Exercise library** — fully custom: name, day tag (Push/Pull/Legs/Cardio/Any), sets, reps, weight, rest-between-sets, rest-before-next-exercise
- **Live workout logging** — start a session, add exercises, log actual reps/weight per set
- **Automatic rest timers** — countdown ring, sound + vibration on completion, +15s / skip controls
- **Weight Log, Workout Log, Nutrition Log, Supplement & Habits Log, Program, Stretch Routine** — companion Excel tracker (`Recomp_Tracker.xlsx`) for everything outside live gym logging
- **History** — every finished session, sets logged, per-exercise detail
- **Progress tab** — all-time totals (lbs moved, sessions, avg duration), bar charts for weight moved & duration (last 10 sessions), auto-generated top-set progression charts for most-logged exercises
- **Duration + total weight tracking** — captured automatically per session
- **Settings** — editable display name, text-based backup/restore, storage usage readout, full reset
- **AI Personal Trainer** — in-app call to the Claude API: input equipment + goal + days/week, get a real generated routine, review and selectively import into the library
- **Onboarding** — first-open name prompt, sets expectation that data is private per person

---

## 3. Roadmap

### Now (highest value, lowest effort — build next)
- [x] **Workout templates** — save a named session (e.g. "Push Day A") as a one-tap preset instead of re-adding exercises every time *(shipped)*
- [x] **Personal record tracking** — auto-detect and flag when a set beats a previous best (weight or reps) *(shipped)*
- [x] **Plate calculator** — type a target weight, see which plates load per side *(shipped)*

### Next
- [ ] Auto-progression suggestions ("hit the top of your range last time — try +5 lb")
- [ ] Warm-up set calculator (scales 2-3 warm-up sets up to working weight)
- [ ] Supersets/circuits (group exercises with zero rest between them)
- [ ] Body measurement tracking (waist/arm/chest — complements scale weight for recomp)

### Later / needs real infrastructure first
- [ ] Training streaks / light gamification
- [ ] Leaderboard across friends (needs the multi-user piece proven first)
- [ ] Garmin sync (needs a real backend + OAuth — not possible inside an artifact)
- [ ] Push notifications (needs a native app or PWA with proper service workers)
- [ ] Progress photos (needs real file storage — current 5MB text-only storage isn't built for images)

---

## 4. Legal / Naming Notes

*(Not legal advice — a real IP attorney is cheap insurance before any public launch.)*

- **Concept is not protectable.** "An app that logs sets/reps/weight/rest" is a category used by many apps (Strong, Hevy, JEFIT, StrongLifts, Fitbod). Building your own version of that category, in your own code and design, is fine.
- **Avoid:** naming it something confusingly similar to an existing app (trademark risk), copying another app's exact visual design pixel-for-pixel (trade dress risk), or copying actual code/assets.
- **Current design is already distinct** — original dark navy/amber palette, own layout, own copy. Good position to be in.
- Revisit this section seriously only at Phase 3+ — not a blocker before then.

---

## 5. Cost Reference (for when Phase 4 becomes real)

| Item | Cost |
|---|---|
| Apple Developer Program | $99/year |
| Google Play developer account | $25 one-time |
| Backend/hosting (Supabase/Firebase) | Free tier to start, ~$10-25/mo once past free limits |
| Domain name | ~$12-15/year |
| PWA distribution (Phase 2/3) | $0 — just a shareable link |

---

## 6. Decision Log

| Date | Decision | Reasoning |
|---|---|---|
| Initial build | Vanilla HTML/CSS/JS single file, no framework | Fastest to iterate inside an artifact, zero build step |
| Initial build | `window.storage` (per-user, personal scope) over shared storage | Keeps each person's data private by default without custom auth |
| Initial build | Dark navy/amber visual identity | Distinct from Strong's UI, gym-friendly dark mode, ties to existing spreadsheet formatting conventions |
| Post-launch | Added Settings tab with text-based backup instead of file download | Uncertain whether native file downloads work reliably in the sandboxed artifact environment; text-paste is safer |
| Post-launch | Added visible save-failure warnings | Originally failed silently — real risk once someone is trusting it with data |
| Feature round 2 | AI Trainer calls Claude directly via the documented Anthropic API-in-artifacts capability | Genuine AI-generated routines, not canned templates — differentiates from every existing tracker app |
| Feature round 3 (Sep 2026) | Confirmed personal-only use; dropped the "Vitals" idea as an in-app tracked feature — steps/weight history handled as a one-off analysis of `STATS_Retired.xlsx` instead | User explicitly doesn't want daily vitals logging inside the app; wants a periodic "read" of what to improve, not another thing to maintain |
| Feature round 3 (Sep 2026) | Built templates, PR tracking, plate calculator — reused existing components (`wt-card`, `wt-lib-card`, `wt-modal`, `wt-ai-preview-item`) rather than introducing new visual patterns | User asked for no new visuals/videos yet — kept the addition purely functional within the established design system |

---

## 7. Open Questions (revisit later, not urgent)

- Final app name (currently placeholder "SETLOG")
- Whether to ever monetize, or keep it free/personal indefinitely
- At what point (if any) to invest in a real backend vs staying artifact-based
- Whether friends-pilot data separation actually works as expected (needs a real test — see Phase 2 trigger)
