# Armoire — Claude Code brief

AI wardrobe stylist PWA for one real daily user (Brody's friend). **Read `PLAN.md` before doing anything — it is the full spec.** This file only sets working rules.

## Build order

Phases P0 → P4 from PLAN.md §7, strictly in order. Each phase has a "Done =" line — that is the acceptance test. Don't start a phase until the previous phase's Done condition is verified working, not just written.

## Non-negotiables

- **Mobile-first.** She uses this as an installed PWA on her iPhone. Design and verify every screen at 375px viewport. Desktop is an afterthought.
- **Design is decided.** Editorial direction per PLAN.md §11; `mockups/1-editorial-photo.html` is the canonical look. Reuse its tokens and typography. Don't invent new UI styles.
- **API-first.** Every mutation goes through the route handlers in PLAN.md §4 — a future native app reuses them. No client-side Supabase writes for business logic.
- **State logic is sacred.** Wear counts, clean↔dirty transitions, and laundry resets get vitest coverage before any UI polish. Bugs here destroy her trust in the app.
- **AI outputs are contracts.** zod-validate everything returned by Claude; reject and retry on invalid. Haiku 4.5 for vision tagging, Sonnet 4.6 for the stylist agent, prompt caching on both.
- **Secrets and photos.** API keys server-side only, never in the client bundle. Her photos live in a private Supabase bucket behind signed URLs and never touch git. RLS on every table.

## Verification

Run the dev server and exercise the actual flow at mobile viewport (screenshot it) before claiming a screen works. "Wrote the code" ≠ done.

## Env

`.env.local` (gitignored): `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `ANTHROPIC_API_KEY`. Ask Brody to fill values himself — never ask him to paste secrets into chat, never commit them.
