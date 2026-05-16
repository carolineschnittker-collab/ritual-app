# Ritual — Architecture Reference

## Repository Structure

```
/
├── index.html              # Main tracker page
├── select-coffee.html      # Currently brewing selector
├── edit-bean.html          # Add new coffee form
├── shot-history.html       # Full shot log
├── ritual-shared.css       # Shared design system
└── sw.js                   # Service worker stub (PWA)
```

No build step. No package.json. No node_modules. Files are served as-is by Vercel.

---

## Data Flow

```
User interaction
      │
      ▼
HTML page (vanilla JS)
      │
      ▼
Supabase JS client (CDN)
      │
      ▼
Supabase REST API (PostgreSQL)
      │
      ▼
Shared cloud database
(same data seen by Caro + Ross)
```

No local state persistence except:
- `localStorage` key `ritual_grind` — saves current grind setting between sessions

---

## Page Relationships

```
index.html  (tracker — home)
    │
    ├── edit pencil ──────────► select-coffee.html
    │                                │
    │                                ├── tap bean → sets current_bean → back to index
    │                                └── + add new coffee ──► edit-bean.html
    │                                                              │
    │                                                              └── save → back to select-coffee
    │
    ├── view all ────────────► shot-history.html
    │
    └── + add new coffee ───► edit-bean.html
```

All pages share the same bottom nav (4 tabs: Tracker / History / Inventory / Brew).
- Tracker → index.html
- History → shot-history.html
- Inventory → dead link (page not built yet)
- Brew → dead link (page not built yet)

---

## Supabase Table Relationships

```
beans ──────────────────────────────────────────────────┐
  id (uuid, PK)                                          │
  name, roaster, origin, roast_level, process            │
  bag_weight, roast_date, date_opened                    │
  descriptors (jsonb []), phase, created_at              │
                                                         │
current_bean (single row, id = 1)                        │
  bean_id (uuid) ────────────────────────────────────────┘
  name, roaster, region, process, roast_level
  bag_weight, roast_date, date_opened
  descriptors (jsonb), updated_at
  (denormalised copy of the active bean for fast reads)

shots
  id (uuid, PK)
  bean_name (text — snapshot at time of shot)
  bean_id (uuid — soft ref to beans, not enforced)
  grind_setting, dose_in, yield_out, duration
  rating (text — raw comma-separated tags)
  summary_label (text — Balanced / Under-extracted / Over-extracted)
  brewer (Caro / Ross)
  grind_speed (fast / normal / slow)
  rpm (integer, default 1200)
  notes, created_at
```

**Key design decisions:**
- `current_bean` is a denormalised single-row table. When a user selects a coffee in select-coffee.html, we write a copy of the relevant fields there. This means index.html only needs one fast query to get everything about the current coffee.
- `shots.bean_name` is stored as a snapshot string, not just a foreign key. This means shot history stays accurate even if the bean is edited or deleted later.
- `shots.rating` stores the raw selected tags as a comma string (e.g. "sweet, bright, syrupy"). `shots.summary_label` stores the derived result (Balanced/Under-extracted/Over-extracted). Both are saved together.

---

## Shots Left Calculation

```
shots_left = floor(bag_weight / 20) - total_shot_count
```

Uses total shot count across all coffees (not per-bean). This is an approximation — dose is assumed to be ~20g per shot. `bag_weight` is stored as text in `beans` but numeric in `current_bean`; always use `parseFloat()`.

---

## Rating → Summary Label Logic

```
Selected tags grouped by colour:
  acid group:     sour, salty, thin     → Under-extracted
  positive group: sweet, bright,
                  syrupy, balanced      → Balanced
  dark group:     bitter, ashy, dry    → Over-extracted

Whichever group has the most selected tags wins.
Tie-break order: positive > acid > dark
```

---

## CSS Architecture

Single shared stylesheet (`ritual-shared.css`) loaded by every page. No Tailwind. No preprocessor.

**Token system** — all colours/spacing defined as CSS custom properties on `:root`.

**Component classes** — layout primitives (`.page`, `.card`, `.header`, `.bottom-nav`), form elements (inputs, selects, textareas all styled globally), buttons (`.btn-primary`, `.btn-secondary`), tags (`.tag`, `.tag-default`, `.tag-acid`, `.tag-positive`, `.tag-dark`), modals (`.modal-overlay`, `.modal-sheet`), toasts.

**Page-level styles** — each HTML file has its own `<style>` block for page-specific components (e.g. `.bean-card` in select-coffee, `.shot-card` in shot-history, `.combo-wrap` dial in index).

**iOS Safari notes** — all inputs require `-webkit-appearance: none !important` and explicit background colour to override Safari's native white styling. This is already handled in ritual-shared.css globally.

---

## PWA Setup (current state)

- `sw.js` exists as a stub but is not fully implemented
- No manifest.json yet
- Can be added to iPhone home screen via Safari Share → Add to Home Screen
- Works offline only for cached pages (not implemented yet)

**To fully implement PWA later:**
1. Add `manifest.json` with app name, icons, theme colour
2. Implement `sw.js` with cache-first strategy for HTML/CSS/JS
3. Add `<link rel="manifest">` to all pages

---

## Pages Not Yet Built

| Tab | Status | Notes |
|---|---|---|
| Inventory | Not started | Should show all beans with edit/archive/delete |
| Brew | Not started | Purpose TBD |

Both nav tabs currently go nowhere (`href="#"`).

---

## Known Quirks & Decisions

**No user auth** — Caro and Ross share a single Supabase dataset. When logging a shot, a "who pulled this?" modal asks for the brewer name manually. This is stored in `shots.brewer`.

**No per-bean shot count** — shots_left is a rough approximation based on bag weight ÷ 20g across all shots ever logged, not just shots for the current bean. This will drift over time as bags are finished and new ones opened.

**Grind setting is local-only** — stored in localStorage, not Supabase. This means Caro and Ross each have their own last-used grind setting on their own devices.

**Descriptors on beans vs shots** — `beans.descriptors` stores *expected* flavour notes (added when a coffee is entered). `shots.rating` stores *actual* tasting notes per shot. These are separate concepts.

**`current_bean.region` vs `beans.origin`** — the `current_bean` table has a column called `region` while the `beans` table uses `origin` for the same concept. Code must write to `beans.origin` and `current_bean.region` (or just omit region from the current_bean upsert).

---

## How to Work on This

**For UI/design changes:**
- Use Claude Design to iterate visually
- Copy the HTML structure exactly from Claude Design output
- Then wire up Supabase JS by replacing mock data only — never rewrite the HTML structure

**For data/logic changes:**
- Work directly in chat with the HTML files uploaded
- Always cross-check column names against the schema above before writing insert/upsert calls
- Test saves by checking Supabase Table Editor

**For deploying:**
1. Download files from Claude
2. Open locally first (all files in same folder) to verify layout
3. Push to GitHub
4. Vercel auto-deploys in ~30s

**When starting a new chat:**
- Paste `ritual-handoff.md` first
- Upload the specific HTML file(s) you want to work on
- State clearly which of the known issues you're fixing
