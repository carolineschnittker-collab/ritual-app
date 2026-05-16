# Ritual — Project Handoff Context

## What this is
A personal espresso tracking PWA called **ritual**. Built for two users (Caro + Ross) sharing a single dataset — no auth. Deployed via Vercel, sourced from GitHub. Designed to feel like a native iOS dark-mode app.

## Stack
- Plain HTML/CSS/JS — no framework, no build step
- `ritual-shared.css` — shared design system (CSS custom properties, not Tailwind)
- Supabase for all data (cloud, shared between both users)
- Vercel for deployment (auto-deploys on GitHub push)

## Supabase
**Project URL:** `https://pblsuaypnzvbdjbnqghv.supabase.co`
**Anon key:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InBibHN1YXlwbnp2YmRqYm5xZ2h2Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzQ2MDg4NTIsImV4cCI6MjA5MDE4NDg1Mn0.qrHP35jRU5uMmGiHgfxqRjmo7Mg_5aA430Te2rDDGbM`

**Initialise in every file like this:**
```js
const db = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```
CDN: `<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>`

---

## Database Schema (confirmed)

### `beans` table
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| name | text | coffee name |
| roaster | text | |
| origin | text | country only (e.g. Ethiopia) |
| roast_level | text | Light / Medium-Light / Medium / Medium-Dark / Dark |
| process | text | Washed / Natural / Honey / etc |
| bag_weight | text | grams (parseFloat still works) |
| roast_date | text | DD/MM/YYYY or date string |
| date_opened | text | optional |
| descriptors | jsonb | array of flavour note strings |
| phase | text | resting / ready / freezer |
| notes | text | legacy, unused |
| is_active | boolean | legacy |
| grind_setting | numeric | legacy |
| dose_in | numeric | legacy |
| target_yield | numeric | legacy |
| target_time | numeric | legacy |
| created_at | timestamp | |

### `current_bean` table (single row, id = 1)
| Column | Type |
|---|---|
| id | integer |
| name | text |
| roaster | text |
| region | text |
| process | text |
| roast_level | text |
| bag_weight | numeric |
| roast_date | text |
| date_opened | text |
| updated_at | timestamp |
| descriptors | jsonb |
| bean_id | uuid (ref → beans.id) |

### `shots` table
| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| bean_id | uuid | |
| bean_name | text | |
| grind_setting | numeric | |
| dose_in | numeric | |
| yield_out | numeric | |
| duration | numeric | seconds |
| rating | text | comma-separated raw tags e.g. "sweet, bright" |
| summary_label | text | Balanced / Under-extracted / Over-extracted |
| notes | text | |
| brewer | text | Caro or Ross |
| grind_speed | text | fast / normal / slow |
| rpm | integer | default 1200 |
| created_at | timestamp | |

---

## Files

| File | Purpose |
|---|---|
| `index.html` | Main tracker — grind dial, shot timer, log shot form, recent shots |
| `select-coffee.html` | Pick currently brewing coffee from inventory |
| `edit-bean.html` | Add new coffee to inventory |
| `shot-history.html` | Full shot log with search + filter |
| `ritual-shared.css` | All shared styles, tokens, components |

All files are at root level in the GitHub repo.

---

## Design System (ritual-shared.css)

### CSS Custom Properties
```css
--surface:         #0e0d0c
--surface-low:     #171512
--surface-mid:     #1d1b18
--surface-high:    #242220
--surface-highest: #2e2b28
--primary:         #F1DFD3   /* warm cream — main accent */
--on-primary:      #2a1f1a
--secondary:       #C8864A   /* amber */
--secondary-light: #F0BD8B
--on-surface:      #e8e3de
--on-surface-mid:  #a89888
--on-surface-low:  #5e5048
--outline:         #3d3530
--outline-mid:     #504442
--error:           #ffb4ab
--radius-card:     1.25rem
--radius-input:    0.75rem
--radius-tag:      999px
--font:            'Manrope', sans-serif
```

### Key classes
- `.card` — surface-low bg, radius-card, border rgba(255,255,255,0.05), padding 20px
- `.card-sm` — same but padding 14px 18px
- `.page` — max-width 480px, margin auto, padding 80px 20px 96px, flex-col gap 12px
- `.header` — fixed top, 64px, blur backdrop
- `.bottom-nav` — fixed bottom, 4 tabs
- `.nav-tab` — opacity 0.3 inactive; `.nav-tab.active` — bg primary, color on-primary
- `.btn-primary` — full width, bg primary, on-primary text
- `.btn-secondary` — outline style, primary colour
- `.tag` — pill button, `.tag-default`, `.tag-acid`, `.tag-positive`, `.tag-dark`
- `.tag.selected` — bg primary, on-primary text (for all groups)
- `.section-label` — 9px, uppercase, tracking wide, primary colour, opacity 0.75
- `.field-label` — 9px, uppercase, primary colour
- `.modal-overlay` + `.modal-overlay.open` — bottom sheet modal
- `.modal-sheet` — the sheet itself
- `.toast` + `.toast.show` — toast notification
- `.shot-rating-badge` — inline badge for shot result
- `.badge-current` — bg primary, on-primary (used on currently selected bean)

### Rating badge colours (defined in each page's local CSS)
```css
.rating-balanced { background: rgba(180,200,160,0.08); color: #8aaa74; border: 1px solid rgba(138,170,116,0.15); }
.rating-under    { background: rgba(200,170,90,0.08);  color: #a8904e; border: 1px solid rgba(200,170,90,0.15); }
.rating-over     { background: rgba(200,100,90,0.08);  color: #a86058; border: 1px solid rgba(200,100,90,0.15); }
```

---

## Page Details

### index.html
**Sections:**
1. Currently brewing card (compact) — reads `current_bean` table, shows name + shots left. Edit pencil → `select-coffee.html`
2. Pull shot card — grind dial (SVG, draggable), shot timer (0.0s display), form fields
3. Recent shots — last 4 from Supabase, read-only preview
4. Add New Coffee button → `edit-bean.html`

**Form fields for logging a shot:**
- Shot Time (s) — `id="f-time"` — auto-fills when timer stops
- RPM — `id="f-rpm"` — default 1200
- Dose In — `id="f-dose"` — default 20
- Yield Out — `id="f-yield"` — default 40
- Grind Speed tags — `id="grind-speed-tags"` — fast/normal/slow, single select
- Rate Shot tags — acid group (sour/salty/thin), positive group (sweet/bright/syrupy/balanced), dark group (bitter/ashy/dry), multi-select
- Notes — `id="f-notes"` — textarea

**Summary label logic:**
- Most selected tags from acid group → `Under-extracted`
- Most from positive group → `Balanced`
- Most from dark group → `Over-extracted`
- Tie-break: positive > acid > dark

**Shot save flow:** Save Shot button → bottom sheet modal asking "who pulled this shot?" (Caro / Ross) → inserts to `shots` table

**Grind dial:**
- SVG circular dial, draggable by touch and mouse
- Range: MIN_GRIND=1 to MAX_GRIND=40
- Grind value stored in `localStorage` key `ritual_grind`
- Should show a confirm modal ~0.4s after user stops adjusting (currently broken — needs fixing)

**Known issues to fix in new chat:**
1. Page only scales for mobile, not desktop/laptop — needs responsive layout for wider screens
2. Grind confirm modal not appearing after dial adjustment
3. Tapping the centre of the dial should also start/stop the timer
4. Shot history filter chips not working (filter by summary_label)
5. Want to add filter by brewer (Caro / Ross) on history page
6. edit-bean.html: freezer status hint says "thaw before use" — should say "ready to use from frozen"
7. Header menu icon and settings icon are wrong colour and non-functional (should just be removed or made subtle/inert for now)

### select-coffee.html
- Loads all beans from `beans` table
- Shows each as a tappable card with: initials dot, name, roaster, origin/process/roast_level pills, phase badge, shots left
- Tap → upserts `current_bean` (id=1) with name, process, roast_level, bag_weight, bean_id, roaster
- Instantly navigates back to `index.html`
- "Current" badge on active bean
- Always shows "+ add new coffee" button at bottom → `edit-bean.html`
- Shots left = `floor(bag_weight / 20)` minus total shot count

### edit-bean.html
- Saves to `beans` table
- Fields: name (required), roaster (required), origin (select), process (select), roast level (tag buttons: Light/Medium-Light/Medium/Medium-Dark/Dark), bag weight, roast date (required, defaults to today), date opened (optional), expected flavour notes (multi-select tags + custom input), status/phase (Resting/Ready/Freezer)
- Status hints: Resting = "Still degassing — wait a few days", Ready = "ready to pull shots", Freezer = "ready to use" (NOT "thaw before use")
- Roast level saves as text string matching the button label

### shot-history.html
- Loads all shots from `shots` table, ordered by created_at desc
- Search: filters on bean_name, brewer, rating, notes, grind_speed
- Filter chips: all / Balanced / Under-extracted / Over-extracted (matching `summary_label` exactly)
- Needs brewer filter chips added: Caro / Ross
- Delete: calls Supabase delete, animates card out
- Shot card shows: bean name, brewer + date, stat row (grind / time / dose→yield), notes

---

## Workflow
1. Make changes here in chat
2. Download updated files
3. Upload to GitHub repo (replace existing files)
4. Vercel auto-deploys within ~30 seconds
5. Test on live URL or locally by opening files in browser (must have all files in same folder for CSS to load)

**Important:** When previewing artifacts in claude.ai, the CSS won't load (can't fetch external files) so it will look unstyled. Always test by opening locally or on the live Vercel URL.
