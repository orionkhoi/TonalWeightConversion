# TonalPlan Phase 3 — Goal-Driven Workout Creator (Implementation Plan)

## Goal

User describes what they want to train for (e.g. "more distance on my golf drive", "stronger football kick", "general upper-body strength"), selects which Tonal accessories they own, and the app generates a complete, Tonal-native workout — warmup, working blocks, finisher, cooldown — using only exercises their equipment supports, with suggested starting weights pulled from their own logged data.

## The one rule that makes it work

The LLM never invents exercises. It interprets the free-text goal and arranges moves, but every exercise it can choose from comes from a curated database you pass in, already filtered to the user's owned equipment. This is the difference between "AI that hallucinates a barbell snatch on a cable machine" and a workout that's genuinely doable on a Tonal. Enforced via structured output (the model returns exercise IDs, not names).

## Architecture (hybrid)

```
User (goal text + equipment + params)
        │
   [1] Client filters EXERCISES by owned equipment  ──► candidate list (IDs only)
        │
   [2] Client → Supabase Edge Function  (goal, candidates, constraints)
        │            (Edge Function holds the LLM API key — never in frontend)
   [3] Edge Function → LLM  with JSON-schema tool call
        │            system prompt: "select ONLY from provided IDs"
   [4] Structured workout JSON  ◄── validated against candidate IDs
        │
   [5] Client renders: warmup / blocks / finisher / cooldown
                        + starting weight per move (from Phase-2 multipliers)
```

## Step 1 — Data models

### 1a. Equipment (per user)

Store the user's owned accessories. Add to the existing `user_data.data` blob (no schema change needed):

```js
equipment: {
  handles: true,        // Smart Handles — everyone has these
  bar: true,            // Smart Bar
  rope: false,          // Rope attachment
  bench: true,          // Workout Bench
  ankleStraps: false,   // Ankle Straps (newer accessory — unlocks lower-body/kicking work)
  roller: false,        // Foam roller / recovery
  mat: true             // Workout mat (floor work)
}
```

UI: a checklist of accessories in the Workout Creator (and/or Data tab). Default Handles = on.

### 1b. Exercise database (curated, static)

A constant array shipped with the app. Each exercise is tagged so the engine can filter and the LLM can select intelligently. This is the heart of the feature — quality here = quality workouts.

```js
const EXERCISES = [
  {
    id: 'chest_press',
    name: 'Chest Press',
    pattern: 'horizontal_push',      // movement pattern
    primary: ['chest','triceps','front_delt'],
    equipment: ['handles'],          // ALL must be owned to qualify (bench optional -> see optionalEquipment)
    optionalEquipment: ['bench'],    // improves it but not required
    position: 'standing',            // standing | seated | floor | bench
    unilateral: false,
    tonalModes: ['eccentric','spotter','burnout'], // Tonal features that suit it
    goalTags: ['strength','hypertrophy','push','upper'],
    intensity: 'compound'            // compound | accessory | core | power | mobility
  },
  { id:'cable_woodchop_high_low', name:'Woodchop (high → low)', pattern:'rotation',
    primary:['obliques','core'], equipment:['handles'], position:'standing', unilateral:true,
    tonalModes:['chains'], goalTags:['rotation','power','golf','core','sport'], intensity:'power' },
  { id:'pallof_press', name:'Pallof Press', pattern:'anti_rotation',
    primary:['core','obliques'], equipment:['handles'], position:'standing', unilateral:true,
    tonalModes:[], goalTags:['anti_rotation','core','golf','kicking','stability'], intensity:'core' },
  { id:'rdl', name:'Romanian Deadlift', pattern:'hinge',
    primary:['hamstrings','glutes','lower_back'], equipment:['bar'], altEquipment:['handles'],
    position:'standing', unilateral:false, tonalModes:['eccentric','chains'],
    goalTags:['strength','posterior_chain','power','sport'], intensity:'compound' },
  { id:'standing_hip_flexion', name:'Standing Hip Flexion (kick drive)', pattern:'hip_flexion',
    primary:['hip_flexors','quads'], equipment:['ankleStraps'], position:'standing', unilateral:true,
    tonalModes:['chains','burnout'], goalTags:['kicking','football','power','sport'], intensity:'power' },
  { id:'standing_hip_extension', name:'Standing Hip Extension', pattern:'hip_extension',
    primary:['glutes','hamstrings'], equipment:['ankleStraps'], position:'standing', unilateral:true,
    tonalModes:['eccentric'], goalTags:['kicking','glutes','posterior_chain','sport'], intensity:'accessory' },
  { id:'hip_adduction', name:'Standing Hip Adduction', pattern:'adduction',
    primary:['adductors'], equipment:['ankleStraps'], position:'standing', unilateral:true,
    tonalModes:[], goalTags:['kicking','sport','stability'], intensity:'accessory' },
  { id:'split_squat', name:'Bulgarian Split Squat', pattern:'lunge',
    primary:['quads','glutes'], equipment:['handles'], optionalEquipment:['bench'], position:'standing',
    unilateral:true, tonalModes:['eccentric'], goalTags:['strength','single_leg','sport','golf','kicking'],
    intensity:'compound' },
  // ... continue: rows, lat pulldown, overhead press, lateral raise, curls, pushdowns,
  //     squat, goblet squat, calf raise, face pull, reverse fly, cable rotation,
  //     kneeling crunch, standing leg curl, hip abduction, etc.
];
```

Build this out to ~40–60 exercises. Every real Tonal movement you add expands what the generator can do. Tag honestly — the LLM leans entirely on `goalTags`, `pattern`, `intensity`, and `equipment`.

### 1c. Goal profiles (seed hints, optional but recommended)

Give the LLM a head start for common goals so results are sharp even from vague input. These are emphasis hints, not hard rules:

```js
const GOAL_HINTS = {
  golf:     { emphasize:['rotation','anti_rotation','posterior_chain','single_leg','hip_extension'],
              note:'Prioritize torso rotational power, anti-rotation core stability, and hip/thoracic mobility.' },
  kicking:  { emphasize:['hip_flexion','hip_extension','adduction','single_leg','anti_rotation','power'],
              note:'Kicking is hip-flexion speed + hip-extension power + adductor strength. Ankle straps are key; use explosive tempo.' },
  strength: { emphasize:['compound','push','pull','hinge','squat'],
              note:'Compound lifts, lower reps (4–6), longer rest, progressive overload.' },
  hypertrophy:{ emphasize:['compound','accessory'],
              note:'8–12 reps, moderate rest, higher volume, eccentric emphasis.' },
  endurance:{ emphasize:['accessory','core'], note:'Higher reps (15–20), short rest, circuit style.' }
};
```

### 1d. Generated workout (what the LLM returns, what you render/save)

```js
{
  title: 'Golf Power & Rotation',
  goal: 'more distance on my drive',
  durationMin: 45,
  equipmentUsed: ['handles','bench'],
  sections: [
    { type:'warmup', name:'Dynamic Warmup', items:[
        { exerciseId:'cable_rotation', sets:2, reps:12, tempo:'controlled',
          tonalMode:null, load:'light', cue:'Prime the torso, ~40% effort' } ] },
    { type:'block', name:'Rotational Power', rounds:3, restSec:90, items:[
        { exerciseId:'cable_woodchop_high_low', sets:3, reps:8, tempo:'explosive',
          tonalMode:'chains', load:'moderate', cue:'Drive from the hips, fast up-tempo' } ] },
    { type:'finisher', name:'Core Burnout', items:[ /* ... */ ] },
    { type:'cooldown', name:'Mobility', items:[ /* ... */ ] }
  ]
}
```

## Step 2 — The prompt (structured, constrained)

System prompt (Edge Function):

```
You are a strength coach programming workouts for a Tonal digital-resistance trainer.
You will receive: the user's goal, their experience level, target duration, and a
CANDIDATE list of exercises (each with an id, name, pattern, muscles, intensity, and
supported Tonal modes).

Hard rules:
- Use ONLY exercises from the candidate list. Reference them by their exact `id`.
- Never invent exercises or use equipment not implied by the candidates.
- Build a coherent session: warmup → 1–3 working blocks → optional finisher → cooldown.
- Match volume/intensity to the goal and duration. Respect the emphasis hints.
- Assign a Tonal mode (eccentric, chains, burnout, spotter) only from each exercise's
  supported list, and only when it serves the goal (e.g. chains for power, burnout for finishers).
- Balance push/pull and avoid overloading one joint.
Return ONLY the workout as JSON matching the provided schema.
```

User message (assembled by the Edge Function):

```json
{
  "goal": "more distance on my golf drive",
  "experience": "intermediate",
  "durationMin": 45,
  "emphasisHints": ["rotation","anti_rotation","posterior_chain","single_leg","hip_extension"],
  "candidates": [ /* the equipment-filtered EXERCISES, trimmed to id/name/pattern/primary/intensity/tonalModes/goalTags */ ]
}
```

Force structured output — use tool calling / JSON schema so you get valid JSON every time. With the Anthropic API, define a `tool` whose `input_schema` is the workout shape from Step 1d, and set `tool_choice` to that tool. Then validate: drop any `exerciseId` not in the candidate set before rendering (belt-and-suspenders against hallucination).

## Step 3 — Backend: Supabase Edge Function

Keeps your LLM API key server-side. Create `supabase/functions/generate-workout/index.ts`:

```ts
import { serve } from 'https://deno.land/std/http/server.ts';

serve(async (req) => {
  // Require an authenticated Supabase user (verify the JWT from the Authorization header).
  const { goal, experience, durationMin, emphasisHints, candidates } = await req.json();

  const res = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': Deno.env.get('ANTHROPIC_API_KEY')!,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json'
    },
    body: JSON.stringify({
      model: 'claude-sonnet-5',
      max_tokens: 2000,
      system: SYSTEM_PROMPT,                    // from Step 2
      tools: [{ name:'return_workout', input_schema: WORKOUT_SCHEMA }],
      tool_choice: { type:'tool', name:'return_workout' },
      messages: [{ role:'user', content: JSON.stringify({ goal, experience, durationMin, emphasisHints, candidates }) }]
    })
  });
  const out = await res.json();
  const workout = out.content.find((c:any) => c.type === 'tool_use')?.input;
  return new Response(JSON.stringify(workout), { headers:{ 'content-type':'application/json' }});
});
```

Set the secret: `supabase secrets set ANTHROPIC_API_KEY=sk-...` and deploy: `supabase functions deploy generate-workout`.

Provider-agnostic: swap the fetch for OpenAI (`response_format: json_schema`) or any model — only the Edge Function changes.

## Step 4 — Client: the Workout Creator tab

Add a 5th tab. Flow:

```js
async function generateWorkout(){
  const goal = document.getElementById('goalText').value.trim();
  const durationMin = +document.getElementById('goalDuration').value || 45;
  const experience = document.getElementById('goalExp').value;

  // 1. filter exercises to owned equipment
  const eq = state.data.equipment || { handles:true };
  const owned = k => k==='handles' ? true : !!eq[k];
  const candidates = EXERCISES.filter(x =>
    x.equipment.every(owned) || (x.altEquipment && x.altEquipment.every(owned))
  ).map(trimForPrompt);

  // 2. detect a goal hint keyword (golf/kicking/strength/...) to pass emphasis
  const hint = detectGoalHint(goal);           // simple keyword match -> GOAL_HINTS
  const emphasisHints = hint ? GOAL_HINTS[hint].emphasize : [];

  // 3. call the Edge Function (auth via the current Supabase session token)
  setGenStatus('Generating…');
  const { data: { session } } = await sb.auth.getSession();
  const r = await fetch(`${SUPABASE_URL}/functions/v1/generate-workout`, {
    method:'POST',
    headers:{ 'Authorization':`Bearer ${session.access_token}`, 'content-type':'application/json' },
    body: JSON.stringify({ goal, experience, durationMin, emphasisHints, candidates })
  });
  let workout = await r.json();

  // 4. validate: strip any exerciseId not in our candidate set
  const validIds = new Set(candidates.map(c=>c.id));
  workout.sections.forEach(s => s.items = s.items.filter(i => validIds.has(i.exerciseId)));

  renderWorkout(workout);
}
```

### The Phase-2 tie-in (do this — it's the killer detail)

When rendering each exercise, look up its `id`, map to the exercise name, and if the user has personal data or history for it, suggest a starting Tonal weight:

```js
function suggestLoad(exName, loadTag){
  // loadTag: 'light' | 'moderate' | 'heavy' -> % of recent working weight
  const rows = state.data.sessions.filter(s => s.exercise === exName);
  if(!rows.length) return null;
  const recent = rows[rows.length-1].weight;
  const pct = { light:0.55, moderate:0.75, heavy:0.9 }[loadTag] || 0.7;
  return Math.round(recent * pct / 2.5) * 2.5;   // round to 2.5 lb
}
```

So a generated block reads "Woodchop — 3×8, chains mode, ~35 lb (from your history)." That's the moment the converter, the history log, and the generator all pay off together.

## Step 5 — Save & reuse workouts

Add `workouts: []` to `user_data.data` and let users save a generated session:

```js
state.data.workouts.push({ id:'w'+Date.now(), createdAt:Date.now(), ...workout });
saveData();   // syncs to Supabase via your Phase-3 sync layer automatically
```

Show saved workouts in a list; tapping one re-renders it. Later, "Start" could feed straight into the History tab to log the sets.

## Tonal domain reference (for tagging exercises + modes)

* Accessories: Smart Handles (default), Smart Bar, Rope, Bench, Ankle Straps (unlocks standing hip flexion/extension, ab/adduction — essential for kicking & lower-body isolation), Foam Roller, Mat.
* Movement patterns to cover: horizontal/vertical push & pull, squat, hinge, lunge, hip flexion/extension, ab/adduction, rotation, anti-rotation, calf, core flexion.
* Tonal modes → when to program them: Eccentric (extra load on the lowering — strength/hypertrophy), Chains (resistance rises through the range — power, lockout, explosive work), Burnout (auto-drops weight to push past failure — finishers), Spotter (eases load when you stall — heavy compounds), Smart Flex (auto-adjust — general). Only offer a mode an exercise actually supports.

## Sport goal cheat-sheet (seed more into `GOAL_HINTS`)

* Golf: rotational power (woodchops, cable rotation), anti-rotation stability (Pallof), posterior chain (RDL), single-leg strength (split squat), thoracic/hip mobility in warmup.
* Football kicking: explosive hip flexion (ankle-strap kick drive), hip extension (glutes/hamstrings), adductor strength, single-leg stability, anti-rotation core; fast/explosive tempo, chains + burnout.
* General strength: compound push/pull/hinge/squat, 4–6 reps, eccentric + spotter, progressive overload from History.

## Rollout checklist

1. [ ] Add `equipment` + `workouts` to the `user_data` blob; build the equipment checklist UI.
2. [ ] Author the `EXERCISES` database (start ~40 moves, tag carefully) + `GOAL_HINTS`.
3. [ ] Write & deploy the `generate-workout` Edge Function; set `ANTHROPIC_API_KEY` secret.
4. [ ] Define the workout JSON schema; wire tool-calling so output is always valid JSON.
5. [ ] Build the Workout Creator tab: goal text, duration, experience, generate button.
6. [ ] Client-side validate returned IDs against the equipment-filtered candidates.
7. [ ] Render workout with sections + the Phase-2 starting-weight suggestions.
8. [ ] Save/list/reopen workouts (syncs via existing Supabase layer).
9. [ ] Test goals: "golf drive distance" (no ankle straps vs with), "stronger kick", "upper-body strength" — confirm exercises respect owned equipment.

## Rule-based fallback (no API / offline)

If the Edge Function is unreachable, generate locally: score each candidate by overlap between its `goalTags`/`pattern` and the detected goal's `emphasize` list, pick top movements per pattern, and drop them into a fixed warmup→blocks→finisher→cooldown template with default set/rep schemes by intensity. Less nuanced than the LLM, but keeps the feature working and is a good draft-stage starting point before you wire the API.
