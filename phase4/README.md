# TonalPlan Phase 4 — Movement Library, Smarter Generation & Workout Editing

Phase 4 builds on the Phase 3 Workout Creator. It loads the **complete Tonal movement
catalog**, makes generated workouts more varied and goal-aware, and adds full **editing,
swapping, exporting, and saved-state** handling. All changes live in the single-page app
([`../index.html`](../index.html)) and sync per-user via the existing Supabase layer.

---

## 1. Full movement catalog (315 moves)

- Replaced the ~88 hand-built exercises with the **complete 315-movement Tonal catalog**,
  generated programmatically from the official movement spreadsheet
  (`Movement, On/Off Tonal, Body Region, Main Muscle Group, Accessories`).
- Authoritative fields (name, muscle group, body region, accessory) come straight from the
  sheet. Derived fields (movement pattern, intensity, unilateral, suggested Tonal mode) are
  inferred heuristically from the muscle group + name and may be approximate.
- Added **Pilates Straps** as an equipment option (27 moves need it); the sheet's single
  accessory per move is treated as the primary requirement. Bodyweight/"Off Tonal" moves
  (no cable) are always available for warmups, core, and mobility.

## 2. Movement Library (browse everything)

- New **Movement Library** card in the Workout tab: a searchable, scrollable list of all 315
  moves showing pattern · muscles and equipment badges.
- Search matches **name, muscle, movement pattern, and equipment** (e.g. "pilates" → 27,
  "ankle straps" → 9).
- **"Only my equipment"** toggle; moves needing gear you haven't enabled are greyed out.

## 3. Smarter, more varied generation

- **Reads the request text**, not just 8 broad categories: words like *chest, glutes, core,
  rotation, explosive* map to muscles/patterns, and scoring also weighs each move's primary
  muscles — so different requests produce genuinely different, on-target workouts.
- **Variety per run:** a wider random tiebreaker means regenerating the same goal yields a
  fresh plan instead of an identical one.
- **Muscle diversity:** capped at 2 moves per primary muscle, so a workout isn't three
  near-identical presses (and workouts differ run-to-run).

## 4. Workout editing (⋯ menu)

A **⋯ menu** on each workout with:

- **Change name** — the title is an editable field; renaming a saved workout persists
  automatically. Saving **de-duplicates** identical names (appends 2, 3, …).
- **Change description** — set a custom description (defaults to `goal · duration · equipment`).
- **Replace a move** — enter replace mode, tap ⇄ on any movement, and pick from up to 8
  **similar-muscle alternatives** (filtered to owned equipment, excluding moves already used).

## 5. Export & saved state

- **Copy for Tonal** — exports a plain-text build sheet (copied to clipboard + shown in a
  selectable box) so you can re-create the workout in Tonal's own Custom Workouts. There is no
  public Tonal API to push workouts directly, so this is the intended bridge.
- **Saved state** — after saving, the card shows a **✓ Saved** badge and the Save button is
  removed; a logic guard also blocks accidental double-saves.

---

## Data model (unchanged from Phase 3, extended)

```js
state.data = {
  lifts:     { [exerciseName]: [{ tonal, free }] },   // conversion data + weight suggestions
  equipment: { handles, bar, rope, bench, ankleStraps, pilatesStraps, roller, mat },
  workouts:  [ { id, title, description?, goal, durationMin, equipmentUsed, sections, createdAt } ]
}
```

## Known limitations

- Derived tags (pattern / intensity / unilateral / Tonal mode) are heuristic; a move may
  occasionally be mis-categorized — it can be corrected in the `EXERCISES` array.
- The spreadsheet lists one accessory per move, so a few moves (e.g. barbell lifts that also
  need a bench) may be slightly under-specified on equipment.
- Suggested starting weights require a matching entry in **My Lifts**; otherwise the app shows
  the load tier (light/moderate/heavy).

## Commits in this phase

- Movement Library + database expansion (65 → 88 → **315**)
- Request-aware selection + generation variety + muscle diversity
- Editable titles + name de-duplication + Close button removed
- ⋯ menu (rename / description / replace-move), similar-move swaps, Copy for Tonal, saved state
