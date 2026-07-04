# TonalPlan Phase 3 — Cross-Device Login & Sync (Implementation Plan)

## Goal

Replace the local-only profile system with real accounts that sync across devices. Sign in on your phone, log a lift, open your laptop, it's there.

## The core change

Today your data lives in `localStorage` on one browser. That physically can't sync. This plan adds a cloud backend that holds accounts + data, which every device reads from and writes to.

Chosen stack (defaults — swap if you prefer):

* **Supabase** — hosted Postgres + auth, generous free tier, real security via Row-Level Security. Frontend stays vanilla JS.
* **Email + password** auth (magic link is a 1-line swap, noted below).
* **Sync model:** cloud is the source of truth; `localStorage` becomes a per-user offline cache. Writes are debounced and pushed up; reads hydrate the app on login.

Your existing data object is unchanged:

```js
{ lifts: { [exercise]: [{tonal, free, date}] },
  overrides: { [exercise]: number },
  sessions: [{exercise, weight, reps, mode, date, ts}] }
```

It just moves from a `localStorage` key into one `jsonb` column per user.

## Step 1 — Create the Supabase project

1. Go to supabase.com → New project. Note the **Project URL** and **anon public key** (Settings → API).
2. Auth → Providers → Email is on by default. For dev, turn off "Confirm email" (Auth → Providers → Email) so you can test instantly; turn it back on for production.

## Step 2 — Database schema + security (run in SQL Editor)

```sql
-- One row per user, holding the whole app-state blob.
create table public.user_data (
  user_id    uuid primary key references auth.users(id) on delete cascade,
  data       jsonb not null default '{"lifts":{},"overrides":{},"sessions":[]}'::jsonb,
  updated_at timestamptz not null default now()
);

-- Row-Level Security: each user can only touch their own row.
alter table public.user_data enable row level security;

create policy "read own"   on public.user_data for select using (auth.uid() = user_id);
create policy "insert own" on public.user_data for insert with check (auth.uid() = user_id);
create policy "update own" on public.user_data for update using (auth.uid() = user_id);

-- Auto-create an empty row whenever a new account signs up.
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer as $$
begin
  insert into public.user_data (user_id) values (new.id)
  on conflict do nothing;
  return new;
end; $$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();
```

RLS is the security boundary — the anon key is safe to ship in the frontend because these policies stop any user from reading another's row.

## Step 3 — Add the Supabase client to the page

In `<head>` (or bundle it if you're using a build tool):

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

At the top of your script:

```js
const SUPABASE_URL = 'https://YOUR-PROJECT.supabase.co';
const SUPABASE_ANON = 'YOUR-ANON-KEY';
const sb = supabase.createClient(SUPABASE_URL, SUPABASE_ANON);
```

## Step 4 — Replace the profile system with auth

Remove: the local profile list, PIN modal, `getProfiles/createProfile/openProfile/submitPin/enter/logout` profile logic. Keep your tabs, Convert/Lifts/History/Data views and all their render functions unchanged — they operate on `state.data`, which still exists.

New auth UI (replaces the login screen): email + password fields with Sign in / Sign up buttons.

```js
async function signUp(email, pw){
  const { error } = await sb.auth.signUp({ email, password: pw });
  if (error) return showAuthError(error.message);
  // if email confirmation is ON, tell them to check inbox; else session is live
}
async function signIn(email, pw){
  const { error } = await sb.auth.signInWithPassword({ email, password: pw });
  if (error) return showAuthError(error.message);
}
async function signOut(){ await sb.auth.signOut(); }

// Magic-link alternative (swap for signIn):
// await sb.auth.signInWithOtp({ email, options:{ emailRedirectTo: location.href }});
```

Session-driven boot — this single listener replaces your `init()` auto-resume and drives showing the app vs login:

```js
sb.auth.onAuthStateChange(async (_event, session) => {
  if (session?.user) {
    state.user = session.user;
    await hydrateFromCloud();          // load this user's data
    showApp(session.user.email);       // reveal #app, set header name
  } else {
    state.user = null;
    showLogin();                       // reveal login screen
  }
});
```

## Step 5 — The data layer (cloud + offline cache)

Replace `loadData/saveData` with cloud-aware versions. localStorage is now keyed by user id and used only as an offline cache.

```js
const cacheKey = () => 'tonalplan_cache_' + state.user.id;
const blankData = () => ({ lifts:{}, overrides:{}, sessions:[] });

async function hydrateFromCloud(){
  // 1) instant paint from cache if present
  try { state.data = Object.assign(blankData(), JSON.parse(localStorage.getItem(cacheKey()))); }
  catch { state.data = blankData(); }
  refreshAll();

  // 2) fetch authoritative copy
  const { data, error } = await sb.from('user_data')
    .select('data, updated_at').eq('user_id', state.user.id).single();
  if (!error && data) {
    state.data = Object.assign(blankData(), data.data);
    state.serverUpdatedAt = data.updated_at;
    localStorage.setItem(cacheKey(), JSON.stringify(state.data));
    refreshAll();
  }
  await maybeMigrateLocalData();       // one-time import of old Phase 2 data
}

let saveTimer = null, saving = false;
function saveData(){                    // called everywhere you already call it — signature unchanged
  localStorage.setItem(cacheKey(), JSON.stringify(state.data));  // cache immediately
  clearTimeout(saveTimer);
  saveTimer = setTimeout(pushToCloud, 800);   // debounce network writes
  setSyncStatus('saving');
}

async function pushToCloud(){
  if (!state.user) return;
  if (!navigator.onLine){ setSyncStatus('offline'); return; }
  saving = true;
  const { error } = await sb.from('user_data')
    .update({ data: state.data, updated_at: new Date().toISOString() })
    .eq('user_id', state.user.id);
  saving = false;
  setSyncStatus(error ? 'error' : 'synced');
}

// Retry any pending cache when the network returns
window.addEventListener('online', pushToCloud);
```

Because every existing feature already calls `saveData()` after mutating `state.data`, you get cloud sync for free across Convert overrides, My Lifts, History, and import — no per-feature changes.

## Step 6 — Migrate existing Phase 2 data (one-time)

So current users don't lose their local logs on first cloud sign-in:

```js
async function maybeMigrateLocalData(){
  // Old Phase 2 stored under tonalplan_data_<oldProfileId>. If cloud is empty
  // and a legacy blob exists, offer to import the largest one.
  const empty = !Object.keys(state.data.lifts).length && !state.data.sessions.length;
  if (!empty) return;
  const legacy = Object.keys(localStorage).filter(k => k.startsWith('tonalplan_data_'));
  if (!legacy.length) return;
  const blob = legacy.map(k => { try { return JSON.parse(localStorage.getItem(k)); } catch { return null; } })
                     .filter(Boolean)
                     .sort((a,b) => JSON.stringify(b).length - JSON.stringify(a).length)[0];
  if (blob && confirm('Import your existing local TonalPlan data into this account?')) {
    state.data = Object.assign(blankData(), blob);
    saveData();
  }
}
```

## Step 7 — Sync status + conflict handling

* Add a tiny header indicator: `setSyncStatus('saving'|'synced'|'offline'|'error')` → colored dot + label. Reuse your existing `.note`/`.pill` styles.
* Simple conflict policy (last-write-wins): fine for a single-user-multi-device app. Optional hardening: before `update`, re-fetch `updated_at`; if it's newer than `state.serverUpdatedAt`, merge (sessions are append-only so concatenate + dedupe by `ts`; for `lifts`/`overrides` prefer the incoming edit) and re-push.
* Keep your Export/Import JSON as-is — it's now a manual backup on top of cloud sync, and still your escape hatch.

## Rollout checklist

1. [ ] Create Supabase project; copy URL + anon key.
2. [ ] Run the schema SQL (Step 2); confirm RLS is enabled.
3. [ ] Add supabase-js script + client init (Step 3).
4. [ ] Swap login screen for email/password auth UI (Step 4).
5. [ ] Replace `loadData/saveData` with cloud data layer (Step 5).
6. [ ] Wire `onAuthStateChange` to show app vs login; delete old profile/PIN code.
7. [ ] Add one-time migration (Step 6) and sync-status indicator (Step 7).
8. [ ] Test: sign up on device A, log a lift, sign in on device B → data appears.
9. [ ] Turn email confirmation back ON before sharing publicly.
10. [ ] Host the static file anywhere (Netlify/Vercel/GitHub Pages) — it talks to Supabase over HTTPS.

## What stays exactly the same

All conversion math (verified study formula + quadratic inverse + Epley), confidence scoring, editable multipliers, the SVG history chart, progressive-overload suggestions, and export/import. Only the storage/identity layer changes.

## If you'd rather not use Supabase

* **Firebase:** same shape — `user_data/{uid}` doc in Firestore, security rules `allow read, write: if request.auth.uid == uid;`, `signInWithEmailAndPassword`. Everything above maps 1:1.
* **Your own Node/Postgres:** endpoints `POST /auth/signup`, `POST /auth/login` (bcrypt + JWT), `GET /data`, `PUT /data` (auth-guarded, `where user_id = req.userId`). Same table, same debounced client push. More control, you maintain hosting + auth security.
