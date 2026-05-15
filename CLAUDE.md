# CLAUDE.md — OT Claims Tracker

## What This Is

A single-page web app for tracking overtime (OT) pay claims. Workers log shifts, the app calculates pay automatically using fixed rates, and claims are tracked through a workflow: Pending → Sent → Acknowledged → Paid. All data is stored per-user in Supabase.

---

## Architecture

**Single-file SPA.** The entire application lives in `index.html`:
- Inline `<style>` — all CSS
- Inline `<script>` — all JavaScript
- No build system, no npm, no bundler, no TypeScript
- No separate JS/CSS files

**Backend: Supabase**
- Auth: email/password via `supabase.auth`
- Database: `claims` table with Row Level Security (each user sees only their own rows)
- CDN-loaded SDK: `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2`

**Fonts:** Google Fonts — Barlow Condensed (headings/mono-style labels) and Barlow (body)

---

## Development Workflow

There is **no build step**. Edit `index.html` directly and open it in a browser. To test with real data you need a Supabase project (credentials are embedded in the file — see CONFIG section at the top of the `<script>` block).

```
# All development happens by editing index.html
# No npm install, no build, no compile step
```

When making changes:
1. Edit `index.html`
2. Open in browser (or refresh)
3. Test auth, CRUD, calculations, and export

---

## Supabase Configuration

At the top of the `<script>` block (line ~558):

```js
const SUPA_URL = 'https://mcbavgbmmxxczvgmglrn.supabase.co';
const SUPA_KEY = '...anon key...';
```

The anon key is safe to expose publicly (it's the Supabase `anon` role). Access is controlled by Supabase RLS policies — users can only read/write their own rows (filtered by `user_id`).

---

## Database Schema

**Table: `claims`**

| Column       | Type    | Notes                                   |
|--------------|---------|-----------------------------------------|
| `id`         | text PK | Timestamp string (e.g. `"1715000000000"`) |
| `user_id`    | uuid    | From `supabase.auth.getUser()`           |
| `date`       | date    | ISO format `YYYY-MM-DD`                 |
| `type`       | text    | `"Planned"`, `"Unplanned"`, `"Public Holiday"` |
| `shift`      | text    | `"Day"`, `"Night"`, `"Twilight"` (null for Unplanned) |
| `start_time` | text    | `HH:MM` — only used for Unplanned OT   |
| `extra_hrs`  | numeric | Extra hours worked — only for Unplanned |
| `miles`      | numeric | One-way miles (return calculated automatically) |
| `n`          | numeric | Normal hours                            |
| `u`          | numeric | Unsociable hours                        |
| `base_pay`   | numeric | `(n + u) × rate`                       |
| `unsoc_pay`  | numeric | `u × 2.41`                             |
| `trav_pay`   | numeric | `miles × 2 × 0.45`                     |
| `total`      | numeric | `base_pay + unsoc_pay + trav_pay`       |
| `date_sent`  | date    | When the claim was submitted            |
| `date_ack`   | date    | When the claim was acknowledged         |
| `date_paid`  | date    | When payment was received               |
| `notes`      | text    | Free-text                               |

---

## Naming Convention: camelCase ↔ snake_case

The app uses **camelCase** internally; the database uses **snake_case**. Two functions handle the mapping:

- `appToDB(claim)` — converts app object → DB row before upsert
- `dbToApp(row)` — converts DB row → app object after fetch

**Always go through these functions.** Never write DB column names directly in application logic or vice versa.

| App (camelCase) | DB (snake_case) |
|-----------------|-----------------|
| `startTime`     | `start_time`    |
| `extraHrs`      | `extra_hrs`     |
| `basePay`       | `base_pay`      |
| `unsocPay`      | `unsoc_pay`     |
| `travPay`       | `trav_pay`      |
| `dateSent`      | `date_sent`     |
| `dateAck`       | `date_ack`      |
| `datePaid`      | `date_paid`     |

---

## Business Logic

### Pay Rates (`R` constant)

```js
const R = { planned: 36.13, pubHol: 48.17, unsoc: 2.41, mile: 0.45 };
```

- `planned` — hourly rate for Planned and Unplanned OT
- `pubHol` — hourly rate for Public Holiday OT
- `unsoc` — additional unsociable hours supplement per hour
- `mile` — pence-per-mile mileage rate

### Shift Presets (`SHIFTS` constant)

```js
const SHIFTS = { Day: {n:12, u:0}, Night: {n:2, u:10}, Twilight: {n:7, u:5} };
```

- **Day** — 07:00–19:00: 12 normal hours, 0 unsociable
- **Night** — 19:00–07:00: 2 normal hours, 10 unsociable
- **Twilight** — 13:00–01:00: 7 normal hours, 5 unsociable

These apply only to Planned and Public Holiday OT. Unplanned OT calculates hours dynamically.

### Calculation Flow (`calc` function)

1. **Planned / Public Holiday**: hours come from `SHIFTS[shift]`
2. **Unplanned**: hours are entered manually; `unsocHrs(startTime, extraHrs)` calculates how many of those hours fall in the unsociable window (20:00–06:00)
3. `basePay = (n + u) × rate` (rate depends on type)
4. `unsocPay = u × R.unsoc`
5. `travPay = miles × 2 × R.mile` (miles doubled for return journey)
6. `total = basePay + unsocPay + travPay`

### Unsociable Hours Window

The `unsocHrs(startStr, hrs)` function counts hours between **20:00 and 06:00** (next day). It handles midnight crossover correctly with two overlap ranges: `[20:00–24:00]` and `[00:00–06:00]`.

### Status State Machine

Status is **derived** — never stored separately. The `status(claim)` function returns:

```
pending → sent → acknowledged → paid
```

- `paid` if `datePaid` is set
- `acknowledged` if `dateAck` is set
- `sent` if `dateSent` is set
- `pending` otherwise

---

## Key Functions Reference

| Function | Purpose |
|---|---|
| `signIn()` | Authenticates via Supabase; calls `showApp()` on success |
| `signOut()` | Signs out, resets state |
| `showApp()` | Shows the app UI, initiates `loadClaims()` |
| `loadClaims()` | Fetches all claims for current user, ordered by date desc |
| `upsertClaim(claim)` | Inserts or updates a claim row |
| `deleteClaimFromDB(id)` | Deletes a claim by id |
| `appToDB(claim)` | Maps app object → DB row |
| `dbToApp(row)` | Maps DB row → app object |
| `calc(type, shift, startTime, extraHrs, miles)` | Computes `{n, u, basePay, unsocPay, travPay, total}` |
| `unsocHrs(startStr, hrs)` | Counts hours in the unsociable window |
| `status(claim)` | Derives status string from date fields |
| `render()` | Filters `claims[]` and rebuilds the table + stats |
| `updateStats()` | Updates the 6 stat cards from `claims[]` |
| `openModal(id)` | Opens claim form; `id=null` for new, id string to edit |
| `closeModal()` | Closes form and resets `editId` |
| `typeChanged()` | Shows/hides shift vs. time fields based on OT type |
| `recalc()` | Recalculates preview values in the modal calc-bar |
| `saveClaim()` | Validates, calls `upsertClaim`, updates local state |
| `confirmDelete()` | Shows confirm dialog, then calls `deleteClaimFromDB` |
| `exportCSV()` | Generates CSV; uses download link or export modal on iOS |
| `isIOS()` | User-agent sniff for iOS |
| `initInstallBanner()` | Shows PWA install prompt on iOS Safari (not standalone) |
| `toast(msg, type)` | Shows/hides the bottom toast notification |
| `showConfirm(opts)` | Promise-based confirm dialog |
| `setSyncStatus(state, msg)` | Updates the sync status bar dot + text |

---

## State

Three module-level variables hold all runtime state:

```js
let claims = [];      // Full array of claim objects for current user
let editId = null;    // ID of the claim being edited, or null for new
let currentUser = null; // Supabase user object
```

`claims[]` is the single source of truth. After any mutation (save/delete), the array is updated in-place and `render()` is called to sync the UI.

---

## UI Conventions

### CSS Variables (Design Tokens)

```css
--bg: #0D1117;   /* page background */
--s1: #161B22;   /* surface 1 (cards, table) */
--s2: #1C2330;   /* surface 2 (headers, inputs) */
--b1: #2A3441;   /* border 1 */
--b2: #3A4859;   /* border 2 */
--blue: #2196F3; --blue2: #1565C0;
--accent: #00E5FF;
--green: #00C853; --amber: #FFB300; --red: #F44336;
--t1: #E8EDF3;   /* primary text */
--t2: #8899AA;   /* secondary text */
--t3: #556070;   /* muted text / labels */
--mono: 'Barlow Condensed'; --sans: 'Barlow';
```

### Button Classes

| Class | Use |
|---|---|
| `.btn-primary` | Blue gradient — main actions |
| `.btn-green` | Green gradient — save/confirm |
| `.btn-ghost` | Subtle — secondary actions |
| `.btn-danger` | Red tint — destructive actions |
| `.btn-amber` | Amber tint — warning actions |
| `.btn-full` | Full width (used in modals/confirm) |

### Status Badge Classes

`.bp` (paid/green), `.ba` (acknowledged/blue), `.bs` (sent/amber), `.bn` (pending/muted)

### Table Row Highlight Classes

`.rp` (paid — green tint), `.ra` (acknowledged — blue tint), `.rs` (sent — amber tint)

### Stat Card Color Classes

`.ca` (accent), `.cb` (blue), `.cg` (green), `.cr` (red), `.co` (amber), `.cv` (purple)

---

## Mobile / PWA Notes

- Uses `env(safe-area-inset-*)` for iPhone notch/home-bar padding
- `-webkit-fill-available` for full-height on iOS Safari
- Modals slide up from the bottom on mobile (`≤600px`), centered dialog on desktop
- `initInstallBanner()` shows an iOS-specific install prompt when running in Safari (not standalone)
- iOS can't trigger file downloads — CSV export falls back to the copy-to-clipboard export modal

---

## What to Watch Out For

- **Rates are hardcoded** in `R` and `SHIFTS`. Any pay rate change requires editing those constants and recomputing stored claims (stored values in the DB are pre-calculated and not retroactively updated).
- **`id` is a timestamp string** (`Date.now().toString()`), not a UUID. Collisions are possible if two claims are created in the same millisecond (rare but worth knowing).
- **No input sanitisation beyond basic validation** — the app trusts Supabase's prepared statements to prevent SQL injection; `fQ` search uses `JSON.stringify(c).toLowerCase().includes(fq)` which is fine for client-side filtering but searches all fields including internal ones.
- **RLS is the only access control** — if Supabase RLS is misconfigured, users could see each other's data.
