# CLAUDE.md

## Project
TrustedStays Distribution Run — a single-page arcade browser game built as a
marketing/gamified piece for TrustedStays (UK short-let distribution company).
Player catches guests, collects rate codes/accreditations/policies/PMS items,
and dodges hazards to unlock an "Amadeus GDS surge" — a playful metaphor for
real GDS onboarding requirements.

## Tech stack
- Plain HTML5 Canvas 2D + vanilla JS (no framework, no bundler, no build step)
- CSS in a `<style>` block (arcade-cabinet UI, Google Fonts: Press Start 2P, Rubik)
- Supabase JS client (`@supabase/supabase-js` via CDN) for the shared leaderboard
- ClickUp form embedded in an iframe modal (feedback CTA)
- GitHub Actions cron job to keep the Supabase project awake

## Key files
- `index.html` — the entire game: markup, styles, and game logic in one file
- `.github/workflows/keep-alive.yml` — daily cron hitting Supabase REST endpoints
- `README.md` — one-line project description
- `TrustedStays_GDS_OnePager.pdf` — reference doc, not used by the game

## Environment variables / secrets
- `SUPABASE_URL`, `SUPABASE_ANON_KEY` — hardcoded near the top of the `<script>`
  in `index.html` (client-side anon key, intentionally public)
- Same two values also stored as GitHub Actions secrets, used by
  `keep-alive.yml` to ping `public_leaderboard` and `marketing_consent`

## Rules
- Do not introduce a framework, bundler, or build step — keep it a static
  single HTML file that opens directly in a browser.
- Do not split `index.html` into multiple files/modules unless explicitly asked.
- Preserve the existing brand palette (CSS custom properties in `:root`:
  burnt-orange, peach, navy, mint, amadeus-blue, etc.) — don't swap colors.
- Never modify Supabase table schemas, RLS policies, or Edge Functions, and
  never change `keep-alive.yml`'s schedule/queries without asking first.
- Don't commit real Supabase service-role keys or other secrets — only the
  anon key is meant to be public here.
