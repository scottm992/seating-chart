# Seating Chart Webapp

A single-file HTML webapp for recording where students sit during tests. Designed for phone use, no build step, no backend, no dependencies.

## Files

- `seating.html` — the entire app (HTML + CSS + JS + base64 icon, ~130KB)

That's it. Drop it on any static host or open it directly from the filesystem on a phone.

## Three views

The app has a tab switcher at the top:

1. **Seating** — the original test-day seat-recording grid (assign students to desks, late/absent status, PNG export).
2. **Check-ins** — homework check-in tracking: give a student a mark /4 with a date and note, see their history over time, and spot who's overdue for a touch-base.
3. **Behaviour** — an incident log: record things you want to track (phone use, long bathroom trips, off-task, a call/email home, a positive) against a student, with a date and optional note. Deliberately the *inverse* of Check-ins — students with recent incidents rise to the top, clean students sink to the bottom, and nothing ever nags you to go log one. See the Behaviours section below.

## Usage

Open `seating.html` on a phone. First load shows an empty landing screen — tap "+ Add a new class" to create your first class, then add students manually or via CSV import. Once a class exists, picking it gives you a seat grid matching the classroom layout. Tap an empty desk to assign a student; tap a filled desk for swap/late/absent/clear actions.

URL params: `seating.html?class=20-1` jumps directly into a class. Useful for per-class home-screen bookmarks.

To install on iOS: Safari → Share → Add to Home Screen. On Android: Chrome → ⋮ → Add to Home screen. The embedded icon (a stylized seating grid matching Scott's actual classroom shape) will render on the home screen.

## Architecture

Single HTML file. Three sections in order:
1. `<head>` — metadata, base64 icon links (apple-touch-icon + favicon), embedded CSS
2. `<body>` — landing screen + app screen + modal markup, all in DOM (toggled via display)
3. `<script>` — class data, state management, render functions, event handlers

No frameworks. Vanilla JS, vanilla DOM. Mobile-first CSS with iOS-style aesthetics (system fonts, blur headers, action-sheet modals, safe-area insets).

### Class data (editable rosters)

The app starts with **no classes**. The user creates them via "+ Add a new class" on the landing screen, or by restoring from a backup. Rosters live entirely in `localStorage`; the script itself contains no embedded class data, only a `PALETTE` of colors for new classes.

Per-class roster storage:

```js
// localStorage key: seating_roster_v1_<classKey>
{
  label: 'Math 20-1',
  color: '#007aff',       // accent color for desks, PNG banner, class pill
  students: [{code, name}, ...]   // sorted alphabetically on save
}

// localStorage key: seating_class_keys_v1
['math-20-1', 'math-31', ...]   // ordered list of class keys; controls landing order
```

Helper functions (all in the script):

- `getClass(key)` — read the live class from localStorage (returns null if missing)
- `getAllClassKeys()` — ordered list of class keys (empty array on fresh install)
- `saveClass(key, data)` — write a class back (sorts students alphabetically)
- `addClass(key, label, color)` — append a new class, picks a palette color if not given
- `deleteClass(key)` — removes class + its current chart + saved charts + check-ins + behaviours
- `addStudentToClass(key, name, explicitCode?)` — adds a student. If `explicitCode` is supplied, uses it after collision check; otherwise auto-generates a 6-digit code via `genStudentCode()`. Returns `{ok, code}` or `{ok: false, reason}`.
- `removeStudentFromClass(key, code)` — **hard-delete**; cascades through check-ins, behaviours, current chart, and saved charts so no orphan references remain
- `changeStudentCode(key, oldCode, newCode)` — changes a student's code. Returns `{ok: true}` or `{ok: false, reason: 'duplicate' | 'student-missing' | 'class-missing'}`. On success, cascades the rename through the check-in object's key, the behaviours object's key, the current chart's assignments/late/absent arrays, and every saved chart — atomically, so a student's history follows them when their code changes.
- `renameStudent`, `renameClass`, `recolorClass` — straightforward edits

A student's code is the primary key for all per-student data: it's the value seats are assigned to, the value in `late`/`absent` arrays, and the object key under which both check-ins and behaviours are stored. So renaming a code is a multi-place migration, not a simple field write.

Class keys are short slugs (`math-20-1`, `math-31`, etc.). When the user creates a new class, the key is derived from the label (lowercased, non-alphanumerics → dashes, with a numeric suffix on collision). Once created, the key never changes — only the visible `label` does — so seat assignments and check-ins keep referring to the same identifier.

### Managing rosters (UI)

Two entry points:

1. **Menu → Manage class & roster** (inside the app, scoped to the current class). Each row in the roster shows the student's name and their `#code`. The **Edit** button opens a modal with two fields: name (required) and code (optional — leaving blank on add auto-generates; on edit, leaving the existing value unchanged preserves it). Duplicate codes are rejected with a toast and the modal stays open so the user can fix the value. The **Remove** button hard-deletes (with a confirmation that lists what will be lost — number of check-ins, seat assignment, late/absent flag). "Class settings" sublayer renames the class, changes color, or deletes the class entirely.
2. **+ Add a new class** button at the bottom of the landing screen. Prompts for label and color, then immediately opens the roster manager for the new (empty) class.

CSV import is in the roster manager. The parser is tolerant:

- Header row: looks for `name`/`student`/`student_name` (required) and `code`/`student_code`/`id`/`student_id` (optional)
- No header: a single-column file is treated as names; a two-column file is treated as `code,name`
- Empty rows are skipped; rows with missing names are skipped with a count reported
- Missing codes are auto-generated; duplicate codes (against existing students in the class, or other CSV rows) are also auto-generated to avoid collision
- Quoted fields with embedded commas/quotes are handled

The user can delete every class — including the last one — and end up back at the empty landing screen. There's no "you must have at least one class" guard.

### Empty landing screen

When `getAllClassKeys()` is empty, the landing screen subtitle changes from "Choose a class" to "Get started by adding your first class," and the only options shown are "+ Add a new class" and "⬆ Restore from backup." The Restore option is also surfaced when classes exist, but it's most useful here — a user reinstalling on a new device opens the page, taps Restore, picks their backup JSON, and they're back in business.

### Room layout

The grid envelope is fixed at **6 columns × 7 rows = 42 slots**, but *which* of those slots are real desks is an **editable, global setting** (shared by every class — all classes use the same room). It's stored in `localStorage` key `seating_layout_v1` as a list of active desk ids; on a fresh install it defaults to the original shape (42 minus the four corner cutouts `r1c1`, `r1c2`, `r7c1`, `r7c6` = 38 desks). Front of the room is row 7 (bottom, nearest the teacher); back is row 1.

`SEATS` is the live array of real desk ids, loaded from that key at startup (`let SEATS = loadLayout()`, with `defaultLayout()` as the fallback). Every consumer — the on-screen grid (`renderGrid`), the front-first randomizer (`frontFirstSeats`), and the PNG export — decides whether a slot is a desk by testing `SEATS.includes(id)`, so they all follow the layout automatically. Non-desk slots still occupy their grid cell (rendered hidden) so the 6-wide alignment and aisle gaps are preserved.

**Editing it (UI):** Menu → **Edit room layout** opens a 6×7 editor where each square toggles between desk and empty space, with a live desk count, FRONT/BACK labels for orientation, and a "Reset to default" button. Saving must leave at least one desk. Layout helpers: `allSlotIds()`, `defaultLayout()`, `loadLayout()`, `saveLayout(ids)`; editor functions `openLayoutEditor()`, `renderLayoutEditor()`, `saveLayoutEdits()`, `resetLayoutEdits()`.

**Consequence on save:** if a desk that a student is currently sitting at gets removed, that student is dropped back to *unseated* on the current chart (their code stays in the roster; late/absent flags are untouched) rather than becoming an orphan. Saved chart snapshots are left as-is; an assignment to a since-removed desk simply won't render if such a snapshot is reopened.

The layout is global, but if a class ever moved to a different room, the natural extension is a per-class layout — store the id list inside the roster object instead of the single global key.

### Status model

A student is in exactly one of these states for a given chart:

| State    | Where stored               | Can be seated? | Notes                                             |
|----------|----------------------------|----------------|---------------------------------------------------|
| Seated   | `chart.assignments[seatId] = code` | (already is)   | May also have late flag                           |
| Unseated | (none — derived)           | yes            | Default state                                     |
| Late     | `chart.late: [code, ...]`  | yes            | A **flag**, not a bucket. Can co-exist with seated|
| Absent   | `chart.absent: [code, ...]`| no             | Mutually exclusive with everything else           |

Important: **late is a flag**, not a bucket. A late student can be either seated (shows L badge on their desk) or unseated (appears in the Late section AND in the seat picker with a LATE badge). Absent students can never be seated and never appear in the picker.

Helper functions:
- `isSeated(code)` — student is currently assigned to a desk
- `isLate(code)` — student has the late flag (regardless of seated/unseated)
- `regularUnseatedCodes()` — not seated, not absent, not late (the "Unseated" section)
- `lateUnseatedCodes()` — late AND not seated (the "Late" section; alphabetical)
- `seatableCodes()` — not seated, not absent (the picker list; includes unseated late)

### Charts

A "chart" is a single snapshot of seat assignments + status flags, with a name and timestamps. Shape:

```js
{
  id: 'c_xxxxxxxx',           // random
  name: 'Chart 2026-05-23',
  createdAt: ISO,
  lastModified: ISO,
  assignments: { 'r3c4': 56020, 'r2c1': 75001, ... },  // seatId → student code
  absent: [code, ...],
  late: [code, ...]
}
```

Each class has independent storage:
- `seating_current_v2_<classKey>` — autosaved on every action
- `seating_saved_v2_<classKey>` — array of explicitly-saved snapshots
- `seating_active_class_v2` — last-used class for landing-screen bypass

All storage is `localStorage`, scoped per browser per device. Charts saved on Scott's phone don't appear on his laptop.

### PNG export

Canvas-based, no library. Renders at 2x DPR. Output includes class banner (colored stripe + label), chart name, timestamp, status summary, the seat grid (with L badge on seated late desks), the FRONT/BACK markers, and footer lists of all Late and Absent students.

Filename format: `<class label> <chart name>.png`.

## State machine summary

Tap targets and their resulting actions:

- **Randomize seats button** (above the grid) → places every present (non-absent) student into seats at random, filling **front-first**: the front row (row 7, closest to the teacher) fills first, then back toward row 1. Clears the existing arrangement (confirms first if seats are already assigned). Late flags are preserved (stored separately from assignments), so a late student keeps their L badge wherever they land. If there are more students than seats, the overflow stays unseated and a toast reports it; if fewer, the empty desks are at the back. See `randomFill()`, `frontFirstSeats()`, `shuffled()` (Fisher–Yates).
- **Check in next button** (beside Randomize seats) → surfaces the student you've gone longest without checking in with and opens the check-in entry modal for them directly from the Seating tab — a fast "who's next" prompt. Never-checked-in students are treated as most overdue; ties are broken at random (so it naturally rotates through them as you log each). Recency uses all check-ins, including note-only touch-bases, matching the Check-ins roster sort. See `pickOverdueCheckinCode()` and `startOverdueCheckin()`.
- **Empty desk** → opens picker → tap student → seats them (with late flag preserved if applicable)
- **Filled desk** → opens action sheet:
  - Swap — clears desk, reopens picker for same desk
  - Mark as late / Remove late flag — toggles the flag, student stays seated
  - Mark absent — unseats, removes late flag, adds to absent
  - Clear seat — unseats, keeps current status (late stays late, otherwise becomes unseated)
- **Unseated chip** → Mark late / Mark absent
- **Late chip** (unseated late student) → Move to unseated (remove late flag) / Mark absent instead
- **Absent chip** → Move to unseated / Mark late instead (also unseats by definition since they weren't seated)

## Check-ins (homework tracking)

A homework check-in is a record that a student showed you practice work and got a mark out of 4. It belongs to a student within a class, independent of any seating chart or test day — these accumulate over the whole semester.

### Check-in data shape

Stored per class in `localStorage` key `seating_checkins_v1_<classKey>`, as an object mapping student code → array of entries:

```js
{
  56020: [
    { id, mark: 0..4 | null, date: 'YYYY-MM-DD', note: '...', createdAt: ISO },
    ...
  ],
  ...
}
```

A check-in entry needs **either a mark or a note** (or both). If `mark` is `null`, it's a **note-only entry** — a touch-base you recorded without scoring (e.g. "spoke with student," "absent, check next class"). Note-only entries:
- Do **not** count toward the student's average or appear in the sparkline (those use scored entries only, `mark !== null`).
- **Do** count as a check-in for recency/overdue purposes — a note-only touch-base still resets the "last seen" clock and is included in the total check-in count.
- Show a pencil glyph (✎) instead of a number, in a distinct blue badge.

In the entry modal, the mark picker is optional: tap a number to select it, tap the same number again to clear it (making the entry note-only). Helper functions: `hasMark(entry)`, `scoredCheckins(code)`, `lastScored(code)`.

The roster badge shows the most recent **scored** mark (so the score signal isn't hidden by a later note); if a student has only note-only entries, the badge shows the pencil glyph.

### Check-ins view

- **+ New check-in** → pick any student → choose mark (0–4), date (defaults today), optional note → save.
- **Student roster** sorted with never-checked-in students at the top, then most-overdue (longest since last check-in) descending. Each row shows the last mark (color-coded badge), how long ago, total count, and a mini sparkline of recent marks.
- Tap a row → **history modal**: all that student's check-ins newest-first, with average, plus a button to add another. Each entry is individually deletable.
- **Overdue threshold** is configurable (Menu → Check-in settings), default 14 days. Students past the threshold show a red "Last: N days ago" line.

Helper functions: `studentCheckins(code)` (sorted newest-first), `lastCheckin(code)`, `daysBetween(dateStr, now)`, `relativeDays(days)` (human phrasing), `totalCheckinCount()` (across all classes, for backup awareness).

## Behaviours (incident tracking)

A behaviour is a logged incident or note tied to a student within a class — a phone caught out, a too-long bathroom trip, a communication home, a positive. Like check-ins, behaviours are independent of any seating chart and accumulate over the semester. The skeleton deliberately mirrors check-ins (per-student dated entries with a note, the same cascade plumbing, the same backup inclusion) so it reuses patterns that are already trusted. The one deliberate divergence is **sort direction**, explained below.

### Behaviour data shape

Stored per class in `localStorage` key `seating_behaviours_v1_<classKey>`, as an object mapping student code → array of entries, exactly parallel to check-ins:

```js
{
  56020: [
    { id, typeId: 'phone', date: 'YYYY-MM-DD', note: '...', createdAt: ISO },
    ...
  ],
  ...
}
```

`typeId` references a **category** rather than a /4 mark. Each entry always has a category; the note is optional.

### Categories are global (not per-class)

The category list lives in **one** `localStorage` key, `seating_behaviour_types_v1` — global across every class, the same way `overdueDays` is global while check-ins are per-class. The rationale: your phone/bathroom/called-home rules are the same in every section, so you edit the list once. Each category:

```js
{ id, label, emoji, tone }   // tone ∈ 'positive' | 'neutral' | 'negative'
```

`tone` drives colour everywhere (red for negative, blue for neutral, green for positive) — the badge on the roster, the dots, the tone tag in the manager, the history marks. Seeded defaults (`DEFAULT_BEHAVIOUR_TYPES`): 📱 Phone, 🚻 Long bathroom, 🗣️ Off-task (negative); ☎️ Called home, ✉️ Emailed home (neutral); 🏫 Office referral (negative); ⭐ Positive (positive).

Helper functions: `loadBehaviourTypes()` / `saveBehaviourTypes()`, `typeById(id)` (returns a graceful `{label:'Removed category', tone:'neutral'}` fallback if the category was deleted — see below), `genTypeId(label)`, `toneClass(tone)`. Per-class storage helpers parallel the check-in ones: `loadBehaviours(cls)`, `persistBehaviours()`, `studentBehaviours(code)` (newest-first), `lastBehaviour(code)`, `totalBehaviourCount()`.

### Behaviour view — the recency inversion

This is the meaningful design difference from Check-ins. A check-in roster is a **to-do list**: never-checked-in students float to the top because they're prompting you to act. A behaviour roster is an **incident log**: zero incidents is the goal, so it sorts the other way.

- Students **with** incidents sort **most-recent-first** (the kid you just logged is at the top while it's fresh).
- Students **with no incidents** sink to the **bottom**, alphabetical, shown as a muted "No incidents". Nothing is styled as overdue; nothing nags you to go create an entry.

Each row shows a tone-coloured badge (the most recent category's emoji), "Last: N ago · M logs", and a row of small tone dots for the recent mix. Tapping a row opens the **history modal**: newest-first, a tally header (`3 phone · 1 called home`), every entry individually deletable.

Entry points to log:
- **+ Log behaviour** (top of the Behaviour view) → pick student → pick category → date (defaults today) → optional note → save.
- **Seat-grid quick path**: tap a seated desk → action sheet → **📝 Log behaviour…** → tap one category → done (today's date, no note, instant toast). This is the fast path for the actual moment you catch something during a test, with the student right in front of you. The quick sheet also has an **✎ Add a note instead…** button that opens the full entry modal (same student, today's date) when you want to annotate.

Category management lives at **Menu → Behaviour categories**: add, edit (label/emoji/tone), remove, or reset to defaults.

### Deleting a category is non-destructive

Removing a category only takes it out of the picker — it does **not** delete logged entries that point at it (that would silently erase history). Those entries are kept and render via the `typeById` fallback as "Removed category" with neutral tone. The remove confirmation reports how many existing entries use the category so the decision is informed. "Reset to defaults" likewise leaves logged history untouched.

### Cascade plumbing

Because behaviours are keyed by student code just like check-ins, the same two cascades apply and are extended for it:
- `removeStudentFromClass` strips `behaviours[code]` (hard-delete).
- `changeStudentCode` renames the `behaviours` object key from old to new, atomically alongside the check-in/chart migrations.
- `deleteClass` removes the per-class `seating_behaviours_v1_<cls>` key.
- `enterClass` loads behaviours into `state.behaviours`; `switchClass` clears it; `onRemoveStudent` / `onSaveStudentEdit` refresh it for the active class.

## Backup & restore (data durability)

**Important context:** `localStorage` is not durable storage. A browser cache clear, iOS evicting site data, switching browsers, or reinstalling the home-screen webapp can wipe everything silently. For seating charts (re-created each test) this is a minor annoyance; for a semester of homework check-ins it would be a real loss. Hence the backup system.

- **Menu → Back up all data (JSON)** downloads a single file `seating-backup-<timestamp>.json` containing *everything*: settings, the global behaviour categories, the global room layout, the class index, and for every class its **roster**, current chart, saved charts, check-ins, and behaviours. The download timestamp is recorded.
- **Menu → Restore from backup** (also available on the empty landing screen) opens a file picker, validates the file (`app: 'seating-chart'`), confirms with the user, and writes everything back to `localStorage`. The restore is a full replace: existing rosters, charts, check-ins, and behaviours are overwritten by the backup's contents. It is **tolerant of older backups** — a pre-behaviours (version 1) file simply lacks the behaviour keys, and restoring it leaves your current global categories untouched rather than clearing them. If the previously active class no longer exists after restore, the app drops back to the landing screen.
- A **backup reminder banner** appears at the top of the Check-ins *and* Behaviour views if there's any tracked data (check-ins or behaviours) and the last backup was 7+ days ago (or never).

Backup JSON shape:

```js
{
  app: 'seating-chart',
  version: 3,
  exportedAt: ISO,
  settings: { overdueDays },
  behaviourTypes: [ { id, label, emoji, tone }, ... ],   // global category list
  layout: ['r1c3', 'r1c4', ...],                          // global desk layout
  classKeys: ['math-20-1', ...],
  classes: {
    'math-20-1': {
      roster: { label, color, students: [...] },
      current: {...chart},
      saved: [...charts],
      checkins: {...},
      behaviours: {...}
    },
    ...
  }
}
```

Version 2 added `behaviourTypes` (global) and per-class `behaviours`; version 3 added the global `layout`. Restore reads older files fine — any missing top-level key is simply skipped, leaving the current value in place.

Recommended habit: back up after any check-in session or roster edit, save the file to Google Drive or Files. This is the durable copy. The backup is also the path for moving data between devices — back up on phone, restore on tablet (or vice versa).

Storage keys summary:
- `seating_roster_v1_<cls>` — per-class roster (label, color, students)
- `seating_class_keys_v1` — ordered list of class keys present
- `seating_current_v2_<cls>` — autosaved current seating chart
- `seating_saved_v2_<cls>` — saved seating chart snapshots
- `seating_checkins_v1_<cls>` — homework check-ins
- `seating_behaviours_v1_<cls>` — behaviour incident log (per class)
- `seating_behaviour_types_v1` — global behaviour category list `[{id,label,emoji,tone}]`
- `seating_layout_v1` — global room layout (list of active desk ids within the 6×7 envelope)
- `seating_settings_v1` — `{ overdueDays }`
- `seating_last_backup_v1` — timestamp (ms) of last backup
- `seating_active_class_v2` — last-used class

## Future change ideas

- **Cloud sync / auto-backup** — the JSON export/import is the manual durability story. A natural next step (good candidate for the Cowork environment, where Google Drive is connected) is a "Save to Drive / Load from Drive" button, or a scheduled auto-backup, so durability doesn't depend on remembering to export.
- **Per-class room layouts** — the layout is now editable in-app (Menu → Edit room layout) but is global; making it per-class would mean storing the desk-id list inside each roster object instead of the single `seating_layout_v1` key
- **Tie check-ins to seating (optional)** — currently fully separate per the design; a "log a check-in" action could be added to the seat action sheet if desired. (Behaviours already do this — the seat action sheet has a 📝 Log behaviour quick path.)
- **Check-in reminders** — surface overdue students more proactively (e.g. a count badge on the Check-ins tab)
- **Reorderable behaviour categories** — drag-to-reorder in the category manager (deferred from the initial behaviours build; fiddly on mobile, low urgency)
- **CSV export of check-ins / behaviours** for gradebook or documentation import
- **Per-behaviour summaries** — e.g. a count badge on the Behaviour tab, or a class-wide "this week" rollup by category
- **Photo attach** — phone camera snapshot stored alongside a chart or check-in (localStorage too small for raw photos; would need IndexedDB)
- **Per-class home-screen icons** in green/blue/purple to match the class colors

## Deployment

Either GitHub Pages, Netlify drop, or any static host. The file is fully self-contained — drop it anywhere that serves HTML and the URL is the deploy.

For home-screen install: load the deployed URL once, then use the browser's "Add to Home Screen" option. The embedded icon will be used.

## Origin

Built in conversation with Claude in May 2026. Designed for Scott's classroom (6-column layout). Single-file constraint chosen for portability, zero-config hosting, and offline-capable home-screen install.
