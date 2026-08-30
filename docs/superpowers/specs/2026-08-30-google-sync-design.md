# Google framework as the cross-device store

Date: 2026-08-30
Status: approved design, not yet implemented

## Problem

The dashboard is used on two devices: a Windows laptop and an iPhone. Calendar events
already sync, because they live in Google Calendar. Nothing else does.

Google Tasks sync for the task rail was built in `7ce4b3c` and is present in `index.html`
(`tapi`, `ensureList`, `syncGoogleTasks`, `pullTasks`, plus write-through in five `wire()`
handlers). It has almost certainly never worked, for two independent reasons:

1. Enabling the **Google Tasks API** in the Cloud Console was a documented prerequisite
   that was never done. Every call returns `403 SERVICE_DISABLED` and fails silently by
   design.
2. `auth/tasks` was added to `SCOPE` in that same commit. If the last consent predates it,
   the cached grant covers Calendar only - and `requestAccessToken({prompt:""})` succeeds
   anyway, returning a token silently missing the Tasks scope.

So the app has been running localStorage-only the whole time.

Three things still do not follow the user between devices:

1. Task sync itself - blocked on both reasons above.
2. Canvas coursework ticks (`S.doneEv`) - deliberately local, per the old design.
3. Due *times* - `pullTasks()` flattens every due date to `T23:59`, because Google Tasks
   discards the time component.

And there are no reminders of any kind.

## Explicitly out of scope

**Preference sync.** Categories, per-category colours, calendar visibility, view and
collapse state stay in localStorage, per device. A new device starts from the seed
defaults. Storing them in Drive `appDataFolder` was designed and then dropped: it was the
only piece requiring an OAuth scope beyond what is already granted, and the alternative -
squeezing them into a Google Tasks note - would put a junk list in the Tasks app on the
phone. Not worth either cost. Revisit only if per-device drift becomes annoying in
practice.

Consequence: **this design requests no new OAuth scope.** `SCOPE` is unchanged.

## Constraints

Inherited from `CLAUDE.md` and non-negotiable:

- Free services only. No paid APIs, no subscriptions.
- Single file. Everything in `index.html` - no build step, no bundler, no npm, no framework.
- No dependencies beyond CDN fonts and Google's GSI script.
- No animation.
- OAuth app stays in Google Cloud **Testing** mode. Do not publish it.

## Approach

Extract a single `// -- Google sync --` section above `wire()`. Today the write-through
logic is inline in five `wire()` handlers as five copies of the same try/revert/toast
shape; this change would add roughly ten more. The section holds `taskPush`/`taskDel`,
`tickPush`/`tickDel`, and `remindUpsert`/`remindDel`, and each `wire()` handler shrinks to
one call.

Rejected: keeping it inline (`wire()` grows past 300 lines, retry logic copy-pasted);
a durable write-behind outbox with offline replay (genuinely more robust on a flaky phone
connection, but a large amount of machinery and a new class of bugs for a single-user
dashboard - YAGNI).

## Failure posture

Unchanged from the existing code and load-bearing for every section below:

- Every Google call is **best-effort**. localStorage is written first and remains the
  offline cache and the fallback.
- On failure: revert the optimistic local change, `toast()` the error, keep working.
- No failure path may drop local data.
- Nothing blocks the UI on the network. Write-through, never write-behind.

---

## Section 0 - Turn the existing sync on

**One manual step, in the Google Cloud Console. Blocks every other section.**

APIs & Services -> Library -> enable **Google Tasks API**. That is the whole Console
change. No new scope is requested, so the OAuth consent screen needs no edit.

**Auth is already shared.** `api()` and `tapi()` go through the same `ensureToken()` and
the same bearer token. There is one sign-in and this design adds none.

**Detect a stale grant.** A cached grant predating `7ce4b3c` lacks `auth/tasks` and will
still satisfy `prompt:""`. Read what Google actually granted from the token response:

```js
const NEED = SCOPE.split(" ");
// in the initTokenClient callback, after r.access_token is confirmed:
if (!google.accounts.oauth2.hasGrantedAllScopes(r, ...NEED)) {
  tokenClient.requestAccessToken({prompt: "consent"});   // re-request, once
  return;
}
```

Self-correcting: consent is forced precisely when a scope is genuinely absent, never
otherwise, and stays correct automatically if `SCOPE` ever changes again. Guard against a
loop by re-prompting at most once per page load.

**Diagnostics readout.** Add `S.svc = {cal:null, tasks:null}`, each value `null` (untried),
`true` (last call succeeded) or an error string, set by the respective subsystem. Render
two monospace pills in the Controls panel - `CAL` and `TASKS` - green on success, red with
the message on failure. The point is that silent failure stops being invisible, which is
what allowed this to go unnoticed since `7ce4b3c`.

---

## Section 1 - Real due times

Google Tasks stores a date and ignores the time. Keep the true timestamp in the notes
field as a second marker line:

```
[Deadline]
[due:2026-09-03T23:59]
any free text notes
```

Generalise `parseTag(notes)` into `parseNotes(notes)` returning
`{tag, due, canvas, text}`. Writers emit the marker block; readers strip it.

**Conflict rule.** On pull, compare the date part of the `[due:...]` marker against Google's
own `due` field:

- Dates agree -> use the marker, preserving the time.
- Dates differ -> **Google wins.** The user edited the date in the Google Tasks app.
  Adopt Google's date at `T23:59` and rewrite the marker to match.
- No marker (task created in the Google Tasks app) -> Google's date at `T23:59`, and write
  a marker on the next push.

---

## Section 2 - Canvas coursework ticks as completed Google Tasks

Ticking a Canvas coursework card creates a **completed** Google Task, so the tick both
follows the user across devices and shows as done in the Google Tasks app.

**Category resolution.** `splitCourse()` already peels the trailing bracket off a Canvas
title, yielding for example `CH 320M/328M`. Split that on `/`, normalise whitespace and
case, and find the first entry of `S.cats` that appears in it. No match -> a list named
`Coursework`, created on demand.

**On tick.** `POST /lists/{listId}/tasks` with:

- `title` = the coursework title (bracket stripped)
- `notes` = `[General]` newline `[canvas:calId|eventId]`
- `status` = `completed`
- `due` = the coursework due date

**On untick.** `DELETE` that task.

**Preventing double entry.** This is the risk flagged when the approach was chosen.
`pullTasks()` must divert any task whose notes carry a `[canvas:...]` marker: it populates
`S.doneEv` and is **excluded** from `S.tasks`. Without this, every tick would come back
down as a second card in the task rail.

**State shape change.** `S.doneEv[key]` goes from a bare ISO string to
`{at, taskId, listId}`. `loadLocal()` coerces legacy string values to `{at: value}` so
nothing already ticked is lost. Two read sites need updating for the new shape:
`evDone()` (truthiness - unaffected, but confirm) and the sort in `doneAssignments()`,
which currently does `new Date(S.doneEv[evKey(y)])` and must become `.at`.

Note `S.doneEv` keeps its localStorage copy as the offline cache, so a tick still works
with the network down and reconciles on the next pull.

---

## Section 3 - Reminders, both paths

### Calendar path - the durable one

A dedicated calendar named **Dashboard Reminders**, created on first run via
`POST /calendars` with `{summary, timeZone: TZ}`, id cached in `bd.remCal`.

Every task tagged `ASAP` or `Deadline` **that has a due date** upserts one event there:

- starts at the due time, 15 minutes long
- `reminders: {useDefault:false, overrides:[{method:"popup",minutes:0},{method:"popup",minutes:60}]}`
  - buzzes an hour before and again at the deadline
- Google's infrastructure delivers it, so the phone buzzes with the dashboard closed.

**A dateless ASAP task gets no calendar event.** There is nothing to schedule, a synthetic
"9am today" event would need recreating every morning and would leave stale events behind,
and such a task is already permanently red at the top of the Tasks feed. (This revises the
9am rule from the in-chat design.)

**Idempotent upsert without a stored map.** Supply the event id from the client, derived
deterministically from the task id. Calendar requires ids to be base32hex (`a`-`v`, `0`-`9`),
length 5-1024; hex is a valid subset, so:

```js
const remId = tid => "bd" + [...tid].map(c => c.charCodeAt(0).toString(16).padStart(2,"0")).join("");
```

Task ids are roughly 20-40 characters, giving 40-80 hex characters, well inside the limit.
Collision-free and reversible. Upsert = `POST /calendars/{cal}/events` carrying `id`; on
**409 Conflict** fall back to `PUT /calendars/{cal}/events/{id}`.

**Lifecycle.** Upsert on: add with a due date, due edit, text edit, tag cycled *into*
ASAP/Deadline. Delete on: task deleted, task completed, tag cycled *out* to General, due
date cleared.

**The reminders calendar must not render.** It appears in `calendarList` and would
otherwise surface every reminder as a duplicate card in the Events feed and a duplicate
block in the grid. Filter it out of `S.cals` in `loadCals()` by cached id - which also
stops `loadEvents()` ever fetching it - and drop it in `mapEvents()` as well, as
belt-and-braces if the calendar is renamed.

### In-page path - the immediate one

- Permission is requested from an explicit **Enable notifications** button in Controls,
  never auto-prompted on load. Chrome penalises unprompted requests.
- A 60-second timer scans tasks and coursework due within the next hour and fires a Web
  `Notification`.
- Fired keys tracked in `bd.notified` as `{date, keys:[...]}`, reset when the date rolls
  over, so a refresh does not re-buzz.

**Known limitation.** iOS Safari only delivers Web Notifications to an installed PWA
(iOS 16.4+). The `manifest.json` already exists, so adding the dashboard to the Home Screen
enables it. The calendar path is the real cross-device buzzer; this one is a same-session
convenience on the laptop.

---

## Order of work

Section 0 first and alone - until the Tasks API is enabled and any stale grant is
re-consented, none of the rest can be verified, and section 0 is also the point at which
the *existing* task sync starts working for the first time. Then sections 1, 2, 3 in
order; section 2 depends on section 1's `parseNotes()`, and section 3 depends on nothing
but section 0.

## Verification

There is no test harness, and adding one would break the single-file constraint. Each
section carries a manual checklist, run twice: once on the laptop, once in a second browser
profile standing in for the phone.

- Section 0 - both service pills read green; a task added on the laptop appears in the
  Google Tasks mobile app; forced re-consent fires at most once.
- Section 1 - a task due 11:59pm survives a pull with its time intact; editing its date in
  the Google Tasks app makes Google's date win on the next pull.
- Section 2 - a coursework tick in profile A shows ticked in profile B, appears completed
  in the Google Tasks app, and appears exactly once (no duplicate card in the rail).
- Section 3 - a reminder event lands on Dashboard Reminders and is absent from the Events
  feed and the grid; completing the task removes it.

## Files touched

- `index.html` - all implementation.
- `CLAUDE.md` - corrections: Tasks sync is built, not planned; document the notes marker
  format, the `S.doneEv` shape change, the reminders calendar exclusion, and the fact that
  prefs remain deliberately per-device. Stays git-ignored.
