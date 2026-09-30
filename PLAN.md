# Armoire — Full App Plan

AI wardrobe stylist for one real daily user (Brody's friend). She photographs her closet once, then every morning picks a vibe and gets a complete outfit — clothes, shoes, bag, jewelry — assembled by an AI stylist from only the items that are actually clean.

## 1. Product overview

**Platform:** Mobile-first PWA (she adds it to her iPhone home screen). Built API-first so it can later be wrapped with Capacitor (Xcode/sideload path) or rebuilt in Expo against the same backend with zero backend changes.

**Users:** Her (primary), Brody (admin/second account). Multi-user data model with RLS so more friends can join later without rework.

### The three core loops

1. **Onboarding loop** — photograph every item → background auto-removed → AI vision auto-tags it → she confirms tags with one tap → digital closet built.
2. **Daily loop** — open app → pick today's vibe (chips + optional free text) → stylist agent proposes a full outfit from clean items, weather-aware → she accepts (items get marked worn), swaps a single piece, or rerolls.
3. **Laundry loop** — worn items accumulate wear counts → hit their tolerance → flip to dirty and leave the styling pool → she taps "Did laundry" (all or a selected load) → back to clean.

## 2. Feature spec

### 2.1 Closet capture & auto-tagging
- Camera or photo-library upload (web `<input capture>` / getUserMedia).
- Client-side background removal via `@imgly/background-removal` (free, on-device WASM) → clean cutout on transparent/neutral background for the closet grid and outfit collages. Original photo kept too.
- On upload, Claude (Haiku 4.5, vision) returns structured JSON tags validated with zod:
  - `category` (top / bottom / dress / outerwear / shoes / bag / jewelry / accessory)
  - `subcategory` (e.g. "cropped tank", "wide-leg jeans", "hoop earrings")
  - `colors[]` (dominant first), `pattern`, `fabric_guess`
  - `warmth` 1–5, `formality` 1–5, `seasons[]`
  - `style_descriptors[]` (e.g. "y2k", "office-core", "coquette", "minimal")
- Tag-confirmation screen: editable chips, one tap to accept. Zero typing required in the happy path.
- Item fields she can set: notes ("runs small", "itchy"), favorite flag, wear tolerance override.

### 2.2 Clean/dirty lifecycle
- Every item has `wear_tolerance` (wears before dirty). Category defaults:
  - tops, dresses: 1 · pants/jeans/skirts: 3 · sweaters/outerwear: 5 · shoes, bags, jewelry, accessories: ∞ (never dirty)
- Accepting an outfit (or logging one manually) creates `wear` records; counts increment; items at tolerance flip to `dirty` and disappear from the stylist's pool.
- Laundry screen: grid of dirty items → "Did laundry" → wash all, or select the actual load → selected items reset to clean, wear count zeroed. Laundry events logged (enables cost-per-wear / most-worn stats later).
- Underwear/socks intentionally not tracked (noise, no styling value).

### 2.3 AI stylist agent
- **Model:** Claude Sonnet 4.6 with tool use, structured output validated with zod, prompt caching on system prompt + closet context.
- **Persona (system prompt, not fine-tuning):** professional fashion stylist. Encodes real styling craft: color theory (complementary/analogous palettes, neutrals as anchors), silhouette balance (volume on top ↔ fitted bottom and vice versa), occasion norms, layering for weather, accessory restraint rules ("edit, don't pile"), and current-but-timeless sensibility. Written for a female client; tone: warm, confident, hype-woman-meets-pro.
- **Context per request:** her style profile (loves/hates, sizes, no-go rules), today's weather (Open-Meteo, free, no key), chosen vibe + free-text notes, clean closet inventory (compact JSON of tags — not images), last 14 days of outfits (avoid repeats), and feedback history (learn her taste over time).
- **Tools available to the agent:**
  - `get_clean_closet(filters?)` — query items by category/warmth/formality
  - `search_inspiration(query)` — web image search for reference looks (P3)
- **Output (structured):** outfit slots — main pieces (top+bottom or dress), shoes, bag, jewelry (0–3), outerwear (weather-conditional) — each mapped to real item IDs, plus a 2–3 sentence "why this works" and one styling tip (tuck, cuff, layering note).
- **Actions:** Wear it (logs wears, marks dirty) · Swap piece (agent re-picks one slot, rest locked) · Reroll (new outfit, avoids the rejected combo) · Feedback after wearing (loved / fine / not me) feeding future context.
- Suggests 1 outfit at a time (fast, decisive), reroll is cheap.

### 2.4 Find similar online (P3)
- On any item: "Find this online" → SerpAPI Google Lens on the item photo → exact/similar product matches with links. Free tier = 100 searches/mo (plenty for one user). Results cached per item in DB.

### 2.5 Inspiration images (P3)
- With an outfit suggestion, agent can attach 1–2 web reference photos of similar real-world looks (via `search_inspiration`), so she can see the pairing on a person.

## 3. Data model (Supabase Postgres, RLS on everything)

```sql
profiles      (user_id PK → auth.users, display_name, city, style_notes text,
               loves text[], hates text[], sizes jsonb, created_at)

items         (id uuid PK, user_id, photo_url, cutout_url,
               category text, subcategory text, colors text[], pattern text,
               fabric text, warmth int, formality int, seasons text[],
               style_descriptors text[], notes text, favorite bool,
               wear_tolerance int, current_wears int default 0,
               state text check (state in ('clean','dirty')) default 'clean',
               archived bool default false, created_at)

outfits       (id uuid PK, user_id, for_date date, vibe text, free_text text,
               item_ids uuid[], reasoning text, styling_tip text,
               status text check (status in ('suggested','accepted','rejected')),
               feedback text check (feedback in ('loved','fine','not_me')),
               weather jsonb, created_at)

wears         (id uuid PK, user_id, item_id → items, outfit_id → outfits null,
               worn_on date)

laundry_events(id uuid PK, user_id, done_at timestamptz, item_ids uuid[])

similar_cache (item_id PK → items, results jsonb, fetched_at)
```

Photos: Supabase Storage, **private bucket**, signed URLs. Photos never touch git.

## 4. API surface (Next.js route handlers — the contract a future native app reuses)

| Route | Purpose |
|---|---|
| `POST /api/items` | upload photo → bg removal happened client-side → store → Haiku tags → return draft item |
| `PATCH /api/items/:id` | confirm/edit tags, notes, tolerance, state, archive |
| `GET /api/closet?state=&category=` | closet grid data |
| `POST /api/outfits/suggest` | `{vibe, free_text?}` → stylist agent → outfit |
| `POST /api/outfits/:id/accept` | log wears, flip dirty items |
| `POST /api/outfits/:id/swap` | `{slot}` → re-pick one piece |
| `POST /api/outfits/:id/feedback` | loved / fine / not_me |
| `POST /api/laundry` | `{item_ids}` → reset to clean |
| `POST /api/items/:id/similar` | Google Lens lookup (cached) |

## 5. Screens

1. **Today (home)** — greeting + weather chip, vibe picker (chips: Work, Casual, Date, Brunch, Cozy, Going Out, Event, Gym + free-text), outfit card with cutout collage, "why this works", actions: Wear it / Swap / Reroll.
2. **Closet** — category tabs, cutout grid, clean/dirty badges, filters (color, formality, season), search, add-item FAB.
3. **Add item** — camera → cutout preview → tag confirmation chips → save. Batch mode for onboarding day.
4. **Item detail** — photo, tags, wear count vs tolerance, state toggle, wear history, "Find this online", edit/archive.
5. **Laundry** — dirty grid, select all/load, "Did laundry", history.
6. **Outfit history** — calendar/list of worn outfits, feedback buttons, re-wear shortcut.
7. **Profile** — style prefs, no-go rules, city, category wear tolerances, sign out.

## 6. Tech stack

- **Framework:** Next.js 15 (App Router) + TypeScript + Tailwind + shadcn/ui
- **PWA:** Serwist service worker, manifest, iOS meta tags; installable from Safari
- **DB/Auth/Storage:** Supabase (free tier) — password or magic-link auth, 2 accounts to start
- **AI:** `@anthropic-ai/sdk` — Haiku 4.5 (vision tagging), Sonnet 4.6 (stylist agent), prompt caching, zod validation on all model outputs
- **Weather:** Open-Meteo (free, no key)
- **Similar search:** SerpAPI Google Lens free tier (P3)
- **Background removal:** `@imgly/background-removal` client-side
- **Hosting:** Vercel (free tier)
- **Native path later:** Capacitor wrap (fastest, keeps web UI, opens in Xcode → sideload/TestFlight) or Expo rebuild against same APIs

## 7. Build phases

- **P0 — Skeleton:** Next.js + Supabase wired, auth, schema + RLS, photo upload to storage, deploy to Vercel. *Done = she can log in and upload a photo.*
- **P1 — Closet:** bg removal, Haiku tagging pipeline, tag confirmation, closet grid, item detail, laundry flow. *Done = full closet digitized, clean/dirty works end to end.*
- **P2 — Stylist:** agent + Today screen, accept/swap/reroll, wear logging. *Done = she gets a wearable outfit for a vibe from clean items only.*
- **P3 — Extras:** similar-item search, inspiration images, outfit history + feedback learning.
- **P4 — Polish:** PWA install flow, app icon, offline closet viewing, empty states, Capacitor spike.

## 8. Testing & verification

- Vitest for laundry/wear state logic (the one place bugs destroy trust).
- Zod schemas as the contract for all AI outputs — reject/retry on invalid.
- Agent eval fixture: a golden closet JSON + 8 vibes → assert structurally valid outfits (no dirty items, no missing slots, weather-sane).
- Playwright e2e: upload → tag → suggest → accept → laundry round trip.

## 9. Security & privacy

- RLS on every table; private storage bucket with signed URLs.
- All API keys server-side only (never in client bundle).
- Her photos stay in Supabase — never in git, never sent anywhere except Anthropic for tagging.

## 10. Running costs

| Thing | Cost |
|---|---|
| Vercel, Supabase, Open-Meteo | $0 (free tiers) |
| One-time closet tagging (~150 items, Haiku) | < $1 |
| Daily outfit agent (Sonnet + caching) | ~$1–3/mo |
| SerpAPI Google Lens | $0 (100/mo free tier) |
| **Total** | **~$1–3/mo** |

## 11. UI direction — Editorial ("The Look")

Chosen from three candidates in `mockups/`. **Canonical reference: `mockups/1-editorial-photo.html`** (rendered in `1-editorial-photo.png`) — build to match it. The app reads like a daily issue of her own style magazine: serif masthead, numbered looks, hairline grids, one red accent.

### Tokens

| Token | Value | Use |
|---|---|---|
| `--ivory` | `#F6F4EE` | app background — everything sits directly on this |
| `--ink` | `#141210` | text, primary buttons, masthead rule |
| `--red` | `#C1121F` | the only accent: active tab underline, "in the wash", live counts |
| `--grey` | `#8A857C` | secondary text, captions, metadata |
| `--line` | `#D8D4CA` | hairlines, 1px grid gutters |

**Type:** Bodoni Moda (display serif) — masthead, look numbers ("Look № 3"), item names, editorial captions, italic section heads ("The wash"). Archivo (sans) — tabs, buttons, metadata. Buttons are ink blocks with letterspaced caps, zero border-radius.

### Layout rules

- ARMOIRE masthead letterspaced (.34em) over a 1px ink rule; grey date · weather · city line beneath.
- Vibe picker and closet tabs are plain text links; active = ink with 2px red underline.
- Closet: 3-column grid with 1px `--line` gutters (gap trick over a `--line` background), square photo cells, serif item name + grey status line below each.
- Outfit plate: cutout collage centered on ivory; "why this works" set as serif magazine copy; one italic styling tip.
- Dirty items: image grayscale at ~50% opacity + red italic "in the wash".
- Photos: her real cutouts (transparent PNGs from bg removal) sit directly on the ivory — no cards, no borders, no shadows. (The mockup uses `mix-blend-mode: multiply` only because its placeholders are white-background JPEGs; real cutouts won't need it.)
