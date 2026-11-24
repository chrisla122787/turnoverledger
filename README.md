# TurnoverLedger

A photographic turnover inspection tool for Airbnb / vacation-rental cleaning
companies. Tagline: **"Proof every turnover was done right."** Descriptive
tagline (shown under the name in the app and on the landing page):
**"Professional Cleaning Inspection Reports."** Positioning line:
"Don't just tell your clients the property is ready. Show them."

Cleaners work a room-by-room checklist on their phones, attach photos, log
damage and issues, and produce a branded PDF inspection report the host or
owner can trust — with a clear **Ready for guest: YES/NO** verdict.

Pricing: **$19/month founding-member pricing** for the first 25 customers,
locked in while the subscription stays active. Standard price is **$29/month**
after the founding spots are gone. The checklist and completion summary are
free, and every free user gets **one complete branded PDF report free** (no
credit card, no watermark — see "First report free" below). Pro unlocks photo
documentation, the damage log, unlimited PDF reports, an evidence pack for
every saved turnover, multiple properties with custom checklists, and the
history dashboard.

Plain static HTML/CSS/JS — no build step, no backend.

## Files

| File | What it is |
|---|---|
| `index.html` | Landing page: promise, how it works, free/Pro features, a static sample report, founding pricing, privacy + billing/cancellation notes. |
| `app.html` | The tool: room-by-room turnover checklist, photo documentation (IndexedDB), damage/issue log, branded PDF report (jsPDF via CDN), multiple properties with custom checklists, history dashboard, backup export/import. |
| `README.md` | This file. |

## Before launch

- In `index.html` and `app.html`, replace the `STRIPE_LINK` placeholder hrefs
  with the real Stripe subscription payment link for TurnoverLedger
  ($19/month founding price). There are two in `index.html` (hero + pricing)
  and one in `app.html` (Pro gate card).
- Contact email in the footer is `chrisla122787@gmail.com`.

## Free vs Pro

**Free (no payment, no account):**

- Interactive room-by-room turnover checklist for one property — Living room,
  Kitchen, Bedroom 1, Bedroom 2, Bathroom, Entry, Outdoor — with task
  checkboxes, an N/A toggle per task, a condition note per room, and a
  per-room sign-off: **Ready / Not ready / N/A**.
- Live completion summary: checklist %, tasks handled, rooms signed off, and
  the overall verdict.
- **First report free:** one complete branded PDF inspection report — with
  photos and issue log if the turnover has them, no watermark, no credit
  card. See "First report free" below.

**Pro (subscription):**

- Photo documentation: photos attached per room, stored in IndexedDB and
  shown as thumbnails in the report.
- Damage / issue log: item, room, description, photo, severity
  (Low / Medium / High), and a resolved / reopen control per issue.
- Branded PDF inspection report: company name/logo, property name + address,
  date/time, cleaner name, room-by-room checklist results (done / N/A /
  not done), issue log (resolved issues are marked), embedded photos, the
  "Ready for guest: YES/NO" verdict, and a signature line. This PDF is the
  client-facing deliverable for the host/owner.
- **Evidence pack (ZIP) per saved turnover:** the report PDF, the ORIGINAL
  room photos (`Original_Room_Photos/`), damage/issue photos
  (`Damage_and_Issue_Photos/`), `Inspection_Record.json` (the full turnover
  record), and `Report_Manifest.txt` (report ID, property, date/time,
  cleaner, verdict, file list). Built entirely in the browser (JSZip via
  CDN). Once exported, the saved turnover is flagged
  `evidenceExported` and the history list shows "Evidence pack exported ✓".
  The pack is the user's own backup — TurnoverLedger never stores a copy,
  and the UI says so in plain words.
- Multiple saved properties, each with its own custom checklist
  (add/remove rooms and tasks).
- Turnover history dashboard: past turnovers per property with date, verdict,
  and issue count; totals for turnovers this month and issues found.
- Backup export (JSON, photos included) / import-restore.

## First report free

Free-tier users can generate **one** complete branded PDF report (the same
report Pro users get — photos and issue log included when present, no
watermark). It is tracked by a localStorage counter
(`turnoverledger.freereport.v1`): the first successful PDF download consumes
it, after which the PDF button asks for Pro. This is best-effort only —
there is no registration or server check, so clearing browser data resets
the counter. That is accepted at MVP: the free report is a sampling device,
not a security boundary. Photo capture and the issue log themselves remain
Pro features in the app UI, so a brand-new free user's first report covers
their checklist and verdict; photos/issues appear in it when the turnover
data has them (e.g. after unlocking, or from a restored backup).

## ⚠️ MVP access note (read before launch)

Pro unlocking is **client-side only** for launch validation:

- The "I already subscribed — unlock Pro" button in `app.html` simply sets a
  `localStorage` flag (`turnoverledger.pro.v1 = "1"`) in the visitor's own
  browser. There is **no real payment verification** — no server checks the
  Stripe subscription, and anyone who finds the button can unlock Pro without
  paying.
- This is a deliberate MVP trade-off to validate demand cheaply on a $0
  static stack (the landing page and the app say so in plain words: unlocking
  is on the honor system during opening validation).
- Before scaling past validation, replace this with a real entitlement check
  (e.g. a small serverless function that verifies the Stripe subscription by
  email before issuing an unlock token).

## The turnover logic

All core logic is in pure functions in `app.html`:

- **Checklist template** — a property's checklist is a list of rooms, each a
  list of task labels. `defaultChecklist()` returns the free-tier default:
  7 rooms, 35 tasks. Pro users can customise it per property.
- **Deriving a turnover from a template** — `deriveRooms(template, prevRooms)`
  builds the working checklist from the template, carrying over done states,
  N/A marks, notes, sign-off status, and photo references from a previous
  version of the same rooms/tasks (matched by name/label), so editing a
  checklist mid-stream doesn't wipe progress.
- **Room sign-off** — each room has a `status`: `"pending"` (Not ready),
  `"ready"`, or `"na"`. (The legacy boolean `ready` is kept in sync —
  `ready === (status === "ready")` — and older saved data that only has the
  boolean is normalised on load by `normalizeRooms`.)
- **Completion** — `completion(rooms)` counts **handled** tasks across all
  rooms: a task counts when it is done **or** marked N/A (N/A tasks stay in
  the denominator — the choice made here is "N/A counts as handled", so the
  percentage always reflects the full checklist):
  `pct = round(handled / total × 100)`.
- **Verdict** — `computeVerdict(rooms, issues)` is YES only when **all** of
  these hold:
  1. there is at least one room,
  2. **every** room is signed off as Ready **or** N/A, and
  3. there is **no unresolved High-severity issue** in the log (issues carry
     a `resolved` flag, toggled from the issue list; Medium and Low issues
     never block the verdict).
  Task completion is reported alongside the verdict, not inside it — a room
  can be 100% ticked but still not signed off, and the summary warns until
  every room is signed off and any High issue is resolved.
- **History totals** — `historyTotals(turnovers, now)` counts turnovers in
  the current calendar month and issues found (this month and all time).
- **Photos** — attached photos are downscaled to at most 1280px (JPEG, 0.82
  quality) where the browser supports canvas, then stored in IndexedDB under
  a generated id. Turnover records store only the photo ids. (The evidence
  pack exports the stored originals — i.e. the full-resolution version the
  app holds, which is the downscaled-to-1280px file if the camera original
  was larger.)

Worked example — the default checklist, mid-clean:

- Living room 5/5, Kitchen 6/6, Bedroom 1 5/5, Bedroom 2 4/5, Bathroom 6/6,
  Entry 4/4, Outdoor 3/4 → **33 of 35 tasks handled = 94% complete**.
- If Bedroom 2 and Outdoor are not yet signed off, the verdict is **NO** —
  even at 94% — until the cleaner signs every room off as Ready (or N/A for
  a room that doesn't apply to this stay, e.g. a locked outdoor area).
- If all 7 rooms are signed off but the log holds an unresolved
  High-severity issue (say, a broken lock), the verdict stays **NO** until
  that issue is marked resolved.

## Data & privacy

Everything lives only in the visitor's own browser. Nothing is sent to a
server.

**localStorage:**

| Key | Contents |
|---|---|
| `turnoverledger.settings.v1` | Company name/phone/email + logo (data URL). |
| `turnoverledger.properties.v1` | Saved properties (Pro): name, address, custom checklist template. |
| `turnoverledger.turnovers.v1` | Saved turnover history (Pro): snapshots incl. verdict, issue count, photo ids. |
| `turnoverledger.draft.v1` | The in-progress turnover + current template, so an interruption doesn't lose work. |
| `turnoverledger.pro.v1` | Pro unlock flag (see MVP note above). |
| `turnoverledger.freereport.v1` | Count of free PDF reports used (0 or 1 — see "First report free"). |

**IndexedDB:** database `turnoverledger`, object store `photos` — photo id →
photo data URL. Photos are deliberately **not** kept in localStorage (they
are far too big for its ~5 MB quota). If IndexedDB is unavailable in a
browser, the app falls back to keeping photos in memory only and tells the
user plainly in the "Your data" card that photos will be lost when the tab
closes. The "Your data" card also shows how many photos are stored and,
where the browser exposes it, approximate storage usage — photo libraries
grow fast, so users are told to export backups and clear old turnovers.

The app tells users plainly: "Your data is stored only in this browser on
this device — export a backup regularly." Backup export downloads a JSON
file with settings, properties, turnovers, the current draft, and all
referenced photos (as data URLs); import restores all of it, photos
included. Deleting a saved turnover also deletes its photos from IndexedDB.

Photo-handling guidance (also on the landing page's Privacy section, in
plain words): turnover photos belong to the cleaning business and the
property owner; get the owner's permission before photographing a property
and before sharing reports containing photos; keep guests' personal
belongings out of frame — photograph the room and its condition, not
people's things; and export each turnover's evidence pack promptly into the
business's own storage, because the browser copy is the only copy until
then.

## Deploy

Any static host works — copy the three files (or at least the two HTML files)
to the host as-is:

- **Netlify / Vercel:** drag-and-drop the folder, or connect the repo; no
  build command, publish directory = this folder.
- **Cloudflare Pages:** same — framework preset "None", no build command.
- **GitHub Pages:** fine for demos/preview only — GitHub Pages terms prohibit
  primarily commercial SaaS use, so don't run the paid product from there.

The only external requests the app makes are the jsPDF and JSZip CDN scripts
in `app.html` (cdnjs). If jsPDF fails to load, everything except the PDF and
evidence-pack buttons still works and the user gets a clear message; if only
JSZip fails, the PDF button still works and the evidence-pack button says
so plainly.
