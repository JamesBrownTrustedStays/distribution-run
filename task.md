# Task: Remember player nickname (localStorage)

## Goal
When a player submits their score, remember only their nickname in
`localStorage` so it can pre-fill the name field next time the score
submission form is shown — while keeping email and marketing consent
fully ephemeral (never persisted, never pre-checked).

## Files / line ranges to modify

All changes are in `index.html`. Line numbers below are current as of
this planning pass — re-verify before editing since earlier edits in
the same session can shift them.

1. **Storage key + safe helpers** — near the existing storage helpers,
   right after `storageOK` is defined (`index.html:1981`) and near
   `TUT_KEY` / `tutorialSeen()` / `markTutorialSeen()`
   (`index.html:1958`, `2197-2198`). Add alongside these:
   - A constant, e.g. `const PLAYER_NAME_KEY = 'ts_dr_player_name';`
   - `function loadSavedPlayerName()` — reads the key, returns `''`
     on any failure or if unset.
   - `function saveSavedPlayerName(name)` — writes the key, no-ops on
     empty/whitespace-only input, swallows any thrown error.

2. **Save on submit** — `handleSubmit()`, `index.html:2127-2145`.
   Right after the existing empty-name guard:
   ```js
   const name = (nameInput.value || '').trim().slice(0, 16);
   ...
   if (!name) { ... return; }
   ```
   add a call to `saveSavedPlayerName(name)` immediately after the
   `if (!name)` guard, **before** the `try { ... }` block that does
   `addLocalEntry` / Supabase insert. This must NOT touch
   `emailInput`, `consentInput`, or anything inside the try/catch —
   the leaderboard write and Supabase logic are unchanged.

3. **Pre-fill on form display** — `endGame()`, `index.html:2331-2345`.
   The existing line:
   ```js
   setTimeout(() => document.getElementById('player-name').focus(), 100);
   ```
   is where the score form becomes visible and the name field is
   focused. Before/alongside this, set
   `document.getElementById('player-name').value = loadSavedPlayerName();`
   (only if a saved name exists — leave the field as-is otherwise,
   since `scoreForm.reset()` / the HTML default already leave it
   empty). The field keeps its existing `required`/`maxlength="16"`
   attributes and stays a normal editable `<input>` — no `readonly`,
   no `disabled`.

## Specific behavior expected

- On successful validation inside `handleSubmit()` (name is
  non-empty after `.trim().slice(0, 16)`), persist that exact string
  to `localStorage['ts_dr_player_name']` — **before** the
  Supabase/local-leaderboard try/catch runs, so the nickname is
  remembered even if the Supabase push later fails offline or errors
  out. This is independent of `sbEnabled` — the feature only depends
  on `storageOK`, not on whether the shared leaderboard is reachable.
- The next time `endGame()` shows the score form, the `#player-name`
  input is pre-filled with the stored value (if any).
- The input remains fully editable/focusable/selectable after
  pre-fill — a second person on the same browser can clear it and
  type their own name.
- Every subsequent successful submit overwrites the stored value with
  whatever name was actually used for that submission.

## Edge cases to handle

- **Empty / whitespace-only names**: never write `''` (or a
  whitespace-only string) to `localStorage`. Because `handleSubmit()`
  already blocks empty names before this code runs (`if (!name) return`
  at line ~2139-2145), `saveSavedPlayerName` only ever receives a
  non-empty, trimmed string from that call site — but the helper
  itself must still guard against empty input defensively (e.g. if
  ever called from elsewhere) so it never stores `""`.
- **localStorage unavailable** (incognito, disabled, quota exceeded,
  security exception): both `loadSavedPlayerName()` and
  `saveSavedPlayerName()` must wrap their `localStorage` calls in
  `try/catch` and fail silently — return `''` / no-op respectively.
  Follow the existing pattern used by `tutorialSeen()` /
  `markTutorialSeen()` (`index.html:2197-2198`), which already checks
  `storageOK` before touching `localStorage` at all. Never throw, never
  block form display or score submission.
- **Unicode / emoji names**: no special handling needed —
  `localStorage.setItem`/`getItem` store arbitrary UTF-16 strings
  natively, and the field is a plain `type="text"` input with no
  charset restriction. Just store/read the string as-is.
- **Field stays editable**: do not add `readonly`, `disabled`, or any
  event handler that reverts edits. Pre-fill is a one-time `.value =`
  assignment when the form is shown, not a binding.
- **No pre-tick of consent, no email persistence**: nothing in this
  feature reads from, writes to, or touches `#player-email` or
  `#player-consent`. The existing behavior — `consentInput.checked`
  reset to `false` after every submit (`index.html:2175`) and no email
  ever written to `localStorage` — must remain exactly as-is.
- **`skipSubmit()` / form reset** (`index.html:2188-2194`): calling
  `scoreForm.reset()` clears the visible field back to blank (or to
  the browser's default, since `player-name` has no `value=` attribute
  in the HTML), which is fine — it does NOT clear the stored
  `localStorage` value. The stored nickname is only ever updated by a
  successful validated submit, never cleared by skipping.

## What NOT to change

- Leaderboard writes: `addLocalEntry(entry)`, `insertToSupabase(entry)`,
  `refreshFromSupabase()`, and the `entry` object shape
  (`index.html:2146-2166`).
- Supabase logic generally: `sbEnabled`, `SUPABASE_URL` /
  `SUPABASE_ANON_KEY`, the `marketing_consent` table write
  (`index.html:2040-2044`).
- Brand palette / CSS custom properties in `:root` and the
  `.score-form` styling rules (`index.html:84-91`).
- Email persistence: `#player-email` must never be read from or
  written to `localStorage`. It continues to be cleared after every
  submit (`emailInput.value = ''`, `index.html:2174`) and only ever
  sent to Supabase's `marketing_consent` table when consent + email
  are both present, exactly as today.
- Consent checkbox state: `#player-consent` must always start
  unchecked when the form is shown (no pre-tick), and continues to be
  reset to `false` after every submit (`index.html:2175`). This
  feature does not read or write `player-consent` at all.
- No new dependencies, frameworks, or build step — plain inline
  `<script>` changes only, consistent with `CLAUDE.md`.

## Out of scope / not touched by this task

- `keep-alive.yml` schedule/queries.
- Supabase table schemas / RLS policies / Edge Functions.
- Any other localStorage keys (`TUT_KEY`, high-score tracking, etc.).
