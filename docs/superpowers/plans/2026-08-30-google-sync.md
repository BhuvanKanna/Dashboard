# Google Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make tasks, coursework ticks and due times follow the user between a Windows laptop and an iPhone, and add reminders that fire with the dashboard closed.

**Architecture:** Google Tasks is the task store, Google Calendar is the event store and the reminder delivery channel; localStorage stays as the offline cache and fallback. All writes are write-through: mutate `S`, `render()`, fire the API call in the background, revert and `toast()` on failure. A new `// ── Google sync ──` section above `wire()` holds every remote call so the handlers stay one-liners.

**Tech Stack:** Vanilla JS in a single `index.html`. Google Identity Services (`accounts.google.com/gsi/client`) for OAuth, Google Calendar API v3, Google Tasks API v1, all via `fetch` with a Bearer token. No build step, no npm, no framework.

**Spec:** `docs/superpowers/specs/2026-08-30-google-sync-design.md`

## Global Constraints

- **Free only.** No paid services, no subscriptions, no API costs.
- **Single file.** Everything in `index.html` — HTML, CSS, JS. No build step, no bundler, no npm, no framework. Vanilla JS only.
- **No dependencies** beyond CDN fonts and Google's GSI script.
- **No animation.** No pulsing, drifting, transitions, or entrance effects. Static only.
- **No new OAuth scope.** `SCOPE` stays exactly `"https://www.googleapis.com/auth/calendar https://www.googleapis.com/auth/tasks"`.
- **OAuth app stays in Google Cloud Testing mode.** Do not publish it.
- Orange `--orange: #FF8A3D` is reserved for class meetings. `CLASS_COLOR` must stay in sync with it.
- Every Google call is best-effort. No failure path may drop local data or block the UI.
- Architecture pattern: mutate `S`, call `render()`, which rebuilds the DOM and calls `wire()`.

## Testing Note

This repo has no test framework and cannot get one without breaking the single-file constraint. So:

- **Pure functions** are verified with `console.assert` snippets pasted into DevTools on the running dashboard. These are real pass/fail checks — a silent console means pass, a red assertion means fail.
- **Network and UI behaviour** is verified with scripted manual checks, each with an explicit expected result.
- "Profile B" below means a second Chrome profile signed into the same Google account, standing in for the iPhone.

---

## Task 1: Service diagnostics and stale-grant detection

Must be first. Everything else in this plan is invisible-on-failure by design, and that is exactly how the existing sync went unnoticed since `7ce4b3c`. This task makes failure visible, and its verification step *confirms the diagnosis* that the Tasks API is disabled.

**Files:**
- Modify: `index.html` — `:root` CSS block (~line 129), `S` object (~line 417), `api()`/`tapi()` (~lines 527-540), `initAuth()` (~line 503), `readout()` (~line 810)

**Interfaces:**
- Consumes: nothing.
- Produces: `S.svc` — `{cal: null|true|string, tasks: null|true|string}`. `null` = untried, `true` = last call succeeded, string = last error message. Also: `api()` and `tapi()` now throw an `Error` carrying a numeric `.status` property, which Task 5 relies on to distinguish `409 Conflict`.

- [x] **Step 1: Add the `good` readout style**

In the CSS, immediately after the existing `.readout div.hot b` rule (~line 129):

```css
.readout div.good b{color:var(--ok)}
```

- [x] **Step 2: Add `svc` to the state object**

In the `let S={...}` literal, alongside `doneEv:{},showDone:false`:

```js
  // Per-service health, surfaced in the Controls readout. null = untried, true = last call
  // OK, string = last error. Exists because every Google call here fails silently by design,
  // which is how the Tasks API being switched off went unnoticed for weeks.
  svc:{cal:null,tasks:null},
```

- [x] **Step 3: Record health and expose the status code in `api()` and `tapi()`**

Replace both functions. The only changes are the two `S.svc` writes and `e.status`:

```js
async function api(path,opts={}){
  await ensureToken();
  const r=await fetch(BASE+path,{...opts,headers:{Authorization:"Bearer "+token,
    "Content-Type":"application/json",...(opts.headers||{})}});
  if(r.status===401){token=null;await ensureToken();return api(path,opts)}
  if(!r.ok){let m=r.status+"";try{m=(await r.json()).error.message}catch(e){}
    S.svc.cal=m;const e=new Error(m);e.status=r.status;throw e}
  S.svc.cal=true;
  return r.status===204?null:r.json()}
async function tapi(path,opts={}){
  await ensureToken();
  const r=await fetch(TASKS_BASE+path,{...opts,headers:{Authorization:"Bearer "+token,
    "Content-Type":"application/json",...(opts.headers||{})}});
  if(r.status===401){token=null;await ensureToken();return tapi(path,opts)}
  if(!r.ok){let m=r.status+"";try{m=(await r.json()).error.message}catch(e){}
    S.svc.tasks=m;const e=new Error(m);e.status=r.status;throw e}
  S.svc.tasks=true;
  return r.status===204?null:r.json()}
```

- [x] **Step 4: Render the two service rows**

Replace `readout()`. Note the labels are `CAL·IO` / `TSK·IO`, not `CAL` / `TASKS` — those two labels are already taken by the calendar count and the open-task count in the same block:

```js
function readout(){
  const u=urgentTasks().length,open=S.tasks.filter(t=>!t.done).length;
  return `<div class="readout">
    <div><span>SYS</span><b>NOMINAL</b></div>
    <div><span>CAL</span><b>${S.cals.length}</b></div>
    <div><span>TASKS</span><b>${open}</b></div>
    <div class="${u?"hot":""}"><span>URGENT</span><b>${u}</b></div>
    ${svcRow("CAL·IO",S.svc.cal)}${svcRow("TSK·IO",S.svc.tasks)}</div>`}
function svcRow(lbl,v){
  const cls=v===true?"good":v==null?"":"hot";
  const txt=v===true?"OK":v==null?"—":String(v).slice(0,26);
  return `<div class="${cls}"><span>${lbl}</span><b>${esc(txt)}</b></div>`}
```

- [x] **Step 5: Detect a grant that is missing a scope**

`auth/tasks` was added to `SCOPE` in `7ce4b3c`. A consent granted before that commit still satisfies `requestAccessToken({prompt:""})` and returns a token with no Tasks scope at all. Read what Google actually granted.

Add above `initAuth()`:

```js
// A cached grant predating a SCOPE change still satisfies prompt:"" and hands back a token
// missing the new scope — silently. Ask Google what was really granted instead of guessing.
let scopeRetried=false;
```

Then in the `initTokenClient` callback, insert this as the first statement *after* the `r.error` guard and *before* `token=r.access_token`:

```js
      if(!scopeRetried&&google.accounts.oauth2.hasGrantedAllScopes&&
         !google.accounts.oauth2.hasGrantedAllScopes(r,...SCOPE.split(" "))){
        scopeRetried=true;                       // once per page load, so this cannot loop
        tokenClient.requestAccessToken({prompt:"consent"});
        return}                                  // `pending` stays set; the retry resolves it
```

- [ ] **Step 6: Verify the readout works and confirm the diagnosis**

Hard-refresh the dashboard **with the Tasks API still disabled** and open Controls.

Expected: `CAL·IO` reads `OK` in green. `TSK·IO` reads red with a message containing `disabled` or `has not been used` — this is the `403 SERVICE_DISABLED` that has been silently breaking task sync all along. Seeing it is the point of this task.

- [ ] **Step 7: Enable the Tasks API and confirm it goes green**

**Manual, Google Cloud Console:** APIs & Services → Library → search "Google Tasks API" → **Enable**. Same project as the OAuth client. No consent-screen edit is needed; no new scope is requested.

Wait ~1 minute for propagation, then hard-refresh the dashboard.

Expected: a Google consent screen may appear once (the stale-grant path from Step 5) — accept it. Then `TSK·IO` reads `OK` in green, and your existing four seed tasks appear in the Google Tasks app on your phone under lists named after their categories. **This is the moment task sync starts working for the first time.**

- [x] **Step 8: Commit**

```bash
git add index.html
git commit -m "Surface per-service sync health and detect stale OAuth grants

Every Google call in this file fails silently by design, which is how the
Tasks API being switched off went unnoticed since 7ce4b3c. Adds CAL/TSK
health rows to the Controls readout, threads the HTTP status onto thrown
errors, and uses hasGrantedAllScopes to force one re-consent when a cached
grant is missing a scope."
```

---

## Task 2: Extract the Google sync layer

Pure refactor, no behaviour change. The same try/revert/toast shape is currently copy-pasted across seven call sites; later tasks would add ten more. Doing this first means every task after it edits one function instead of seven.

**Files:**
- Modify: `index.html` — add a section after `pullTasks()` (~line 592); rewrite call sites in `commitTaskEdit()` (~line 716), `commitDueEdit()` (~line 742), `syncGoogleTasks()` (~line 562), and four `wire()` handlers (~lines 1196-1238)

**Interfaces:**
- Consumes: `tapi()`, `listMap`, `ensureList()`, `saveTasks()`, `render()`, `toast()` — all existing.
- Produces:
  - `listPath(listId) -> string`
  - `taskPath(listId, taskId) -> string`
  - `taskBody(t) -> object` — the Google Tasks resource for a local task `t`
  - `taskPatch(t, body, undo)` — fire-and-forget PATCH; calls `undo()` then re-renders on failure
  - `taskPush(t) -> Promise` — POST a new task, rewrites `t.id` to Google's id
  - `taskDel(t, undo)` — fire-and-forget DELETE

- [x] **Step 1: Add the sync layer**

Insert immediately after `pullTasks()`, before `async function loadCals()`:

```js
// ── Google sync ───────────────────────────────────────────────────────────────
// Every function here is best-effort and fire-and-forget. The local write has already
// happened and rendered by the time these are called; a failure calls the supplied undo(),
// re-renders and toasts. Nothing here may block the UI or drop local data.
const listPath=id=>`/lists/${encodeURIComponent(id)}`;
const taskPath=(lid,tid)=>`${listPath(lid)}/tasks/${encodeURIComponent(tid)}`;
// Google Tasks keeps only a DATE in `due` and discards the time component.
function taskBody(t){
  const b={title:t.text,notes:"["+t.tag+"]",status:t.done?"completed":"needsAction"};
  if(t.due)b.due=new Date(t.due.slice(0,10)+"T00:00:00Z").toISOString();
  return b}
function taskPatch(t,body,undo){
  const listId=listMap[t.cat];
  if(!listId)return;                       // not pushed to Google yet — local-only is fine
  tapi(taskPath(listId,t.id),{method:"PATCH",body:JSON.stringify(body)})
    .catch(e=>{undo();saveTasks();render();toast("Sync failed: "+e.message,true)})}
async function taskPush(t){
  const listId=listMap[t.cat]||await ensureList(t.cat);
  const d=await tapi(`${listPath(listId)}/tasks`,{method:"POST",body:JSON.stringify(taskBody(t))});
  t.id=d.id;saveTasks();return d}
function taskDel(t,undo){
  const listId=listMap[t.cat];
  if(!listId)return;
  tapi(taskPath(listId,t.id),{method:"DELETE"})
    .catch(e=>{undo();saveTasks();render();toast("Sync failed, restored: "+e.message,true)})}
```

- [x] **Step 2: Route the migration loop through `taskPush`**

In `syncGoogleTasks()`, replace the body of the `if(t.id.length<20){...}` block with:

```js
        if(t.id.length<20){ // still a local uid(), not yet pushed to Google
          await taskPush(t)}
```

- [x] **Step 3: Route `commitTaskEdit` through `taskPatch`**

Replace the `const listId=listMap[t.cat]; if(listId)tapi(...)` block inside `commitTaskEdit()` with:

```js
    taskPatch(t,{title:val},()=>{t.text=old});
```

- [x] **Step 4: Route `commitDueEdit` through `taskPatch`**

Replace the `const listId=listMap[t.cat]; if(listId){...}` block inside `commitDueEdit()` with:

```js
    taskPatch(t,{due:t.due?new Date(t.due.slice(0,10)+"T00:00:00Z").toISOString():null},
      ()=>{t.due=old});
```

- [x] **Step 5: Route the four `wire()` handlers through the layer**

Tag cycle — replace its `const listId=...; if(listId)tapi(...)` tail with:

```js
    taskPatch(t,{notes:"["+t.tag+"]"},()=>{t.tag=old});
```

Toggle done (`data-tg`) — replace its tail with:

```js
    taskPatch(t,{status:t.done?"completed":"needsAction"},()=>{t.done=was});
```

Remove (`data-rm`) — replace its tail with:

```js
    taskDel(t,()=>{S.tasks.push(t)});
```

Add (`data-add`) — replace the whole trailing `(async()=>{try{...}catch(e){}})();` IIFE with:

```js
      taskPush(t).catch(()=>{}); // stays local-only; picked up on the next successful sync
```

- [ ] **Step 6: Verify no behaviour changed**

Hard-refresh. In the Tasks rail: add a task, rename it, change its tag, set a due date, tick it, untick it, delete it.

Expected: each action behaves exactly as before, `TSK·IO` stays green throughout, and each change appears in the Google Tasks phone app within a few seconds.

- [x] **Step 7: Commit**

```bash
git add index.html
git commit -m "Extract the Google Tasks write-through calls into one sync layer

The same try/revert/toast shape was copy-pasted across seven call sites and
the reminders and coursework-tick work would have added ten more. No
behaviour change."
```

---

## Task 3: Real due times

Google Tasks stores a date and throws the time away, so an 11:59pm deadline currently round-trips back as a bare date. Keep the true timestamp in the notes field.

**Files:**
- Modify: `index.html` — replace `parseTag()` (~line 561), update `taskBody()`, `pullTasks()` (~line 579), the tag-cycle handler, and `commitDueEdit()`

**Interfaces:**
- Consumes: `taskBody`, `taskPatch` from Task 2.
- Produces:
  - `parseNotes(notes) -> {tag, due, canvas, text}` — `tag` is one of `"General"|"ASAP"|"Deadline"`, `due` is a local `YYYY-MM-DDTHH:MM` string or `null`, `canvas` is a `calId|eventId` string or `null`, `text` is the user's free text.
  - `buildNotes({tag, due, canvas, text}) -> string`
  - Task 4 depends on the `canvas` field of both.

- [x] **Step 1: Replace `parseTag` with `parseNotes` and `buildNotes`**

Delete the `parseTag` function and put this in its place:

```js
// Notes carry a small machine header, then the user's own text:
//   [Deadline]
//   [due:2026-09-03T23:59]
//   [canvas:calId|eventId]
//   free text
// The due marker exists because Google Tasks keeps only a DATE in its own `due` field and
// discards the time, so the real timestamp has nowhere else to live. Markers must be
// contiguous at the top; the first non-marker line begins the user's text.
function parseNotes(notes){
  const out={tag:"General",due:null,canvas:null,text:""};
  const lines=String(notes||"").split("\n");
  let i=0;
  for(;i<lines.length;i++){
    const s=lines[i].trim();
    let m;
    if(m=/^\[(ASAP|Deadline|General)\]$/.exec(s)){out.tag=m[1];continue}
    if(m=/^\[due:([\d\-T:]+)\]$/.exec(s)){out.due=m[1];continue}
    if(m=/^\[canvas:(.+)\]$/.exec(s)){out.canvas=m[1];continue}
    break}
  out.text=lines.slice(i).join("\n").trim();
  return out}
function buildNotes(o){
  const h=["["+(o.tag||"General")+"]"];
  if(o.due)h.push("[due:"+o.due+"]");
  if(o.canvas)h.push("[canvas:"+o.canvas+"]");
  if(o.text)h.push("",o.text);
  return h.join("\n")}
```

- [x] **Step 2: Verify the pure functions in DevTools**

Hard-refresh, open the console, paste:

```js
(()=>{
  const p=parseNotes("[Deadline]\n[due:2026-09-03T23:59]\nring the lab");
  console.assert(p.tag==="Deadline","tag");
  console.assert(p.due==="2026-09-03T23:59","due");
  console.assert(p.text==="ring the lab","text");
  console.assert(p.canvas===null,"canvas null");
  const c=parseNotes("[General]\n[canvas:abc|xyz]");
  console.assert(c.canvas==="abc|xyz","canvas parsed");
  console.assert(parseNotes("").tag==="General","default tag");
  console.assert(parseNotes("just prose").text==="just prose","bare prose");
  console.assert(parseNotes("just prose").tag==="General","bare prose tag");
  const r={tag:"ASAP",due:"2026-09-03T23:59",canvas:null,text:"hi"};
  console.assert(JSON.stringify(parseNotes(buildNotes(r)))===JSON.stringify(r),"round trip");
  console.log("parseNotes/buildNotes OK");
})()
```

Expected: `parseNotes/buildNotes OK` and no red assertion failures.

- [x] **Step 3: Emit the marker on every write**

In `taskBody()`, replace the `notes` field:

```js
  const b={title:t.text,notes:buildNotes({tag:t.tag,due:t.due,canvas:t.canvas,text:t.notes}),
    status:t.done?"completed":"needsAction"};
```

- [x] **Step 4: Stop the tag cycle and due edit from wiping the marker**

Both currently PATCH a narrow field set. `notes` holds the tag *and* the due marker now, so a partial write destroys the other one. Both must send the whole rebuilt `notes`.

In the tag-cycle handler, replace the `taskPatch` line from Task 2 Step 5 with:

```js
    taskPatch(t,{notes:buildNotes({tag:t.tag,due:t.due,canvas:t.canvas,text:t.notes})},
      ()=>{t.tag=old});
```

In `commitDueEdit()`, replace the `taskPatch` line from Task 2 Step 4 with:

```js
    taskPatch(t,{due:t.due?new Date(t.due.slice(0,10)+"T00:00:00Z").toISOString():null,
      notes:buildNotes({tag:t.tag,due:t.due,canvas:t.canvas,text:t.notes})},()=>{t.due=old});
```

- [x] **Step 5: Apply the conflict rule on pull**

In `pullTasks()`, replace the `(d.items||[]).forEach(...)` call with:

```js
      (d.items||[]).forEach(it=>{
        const n=parseNotes(it.notes),gdate=it.due?it.due.slice(0,10):null;
        // Google wins on any disagreement — a mismatch means the date was edited in the
        // Google Tasks app, which is the only place that can change it behind our back.
        // The marker only survives when it agrees, and then it contributes the time.
        const due=(n.due&&gdate&&n.due.slice(0,10)===gdate)?n.due:(gdate?gdate+"T23:59":null);
        pulled.push({id:it.id,cat,text:it.title||"(untitled)",tag:n.tag,due,
          notes:n.text,canvas:n.canvas,done:it.status==="completed"})});
```

- [ ] **Step 6: Verify the time survives, and that Google wins a conflict**

1. In the dashboard, set a task's due date. Confirm the rail shows `11:59p`.
2. Hard-refresh (forces a fresh pull). Expected: still `11:59p`, not a bare date.
3. In the Google Tasks phone app, change that task's date to tomorrow.
4. Back in the dashboard, hard-refresh. Expected: the rail shows tomorrow at `11:59p` — Google's date won.

- [x] **Step 7: Commit**

```bash
git add index.html
git commit -m "Keep real due times in a notes marker, since Google Tasks drops the time

Generalises parseTag into parseNotes/buildNotes. Google wins on any date
disagreement, because the Tasks app is the only thing that can change a date
behind our back. Tag cycle and due edit now rewrite the whole notes header,
which they had to before one could clobber the other."
```

---

## Task 4: Canvas coursework ticks as completed Google Tasks

Ticking a coursework card currently writes only to `localStorage`, so a quiz ticked on the laptop still looks pending on the phone.

**Files:**
- Modify: `index.html` — `loadLocal()` (~line 429), `doneAssignments()` (~line 695), `pullTasks()`, the sync layer, the `data-evdone` handler (~line 1200)

**Interfaces:**
- Consumes: `parseNotes`/`buildNotes` (Task 3), `listPath`/`taskPath` (Task 2), `evKey`, `splitCourse`, `isExcluded`, `ensureList` — all existing.
- Produces:
  - `resolveCat(course) -> string` — a category name from `S.cats`, or `"Coursework"`
  - `tickPush(ev) -> Promise` — POSTs a completed task and enriches `S.doneEv[evKey(ev)]`
  - `tickDel(rec) -> Promise` — takes the *record*, not the key, because the caller deletes the entry first
  - `S.doneEv[key]` changes shape from an ISO string to `{at, taskId, listId}`

- [x] **Step 1: Migrate the `doneEv` shape on load**

`S.doneEv[key]` becomes an object. Legacy string values must be coerced so nothing already ticked is lost. In `loadLocal()`, replace the `try{S.doneEv=JSON.parse(...)}catch(e){S.doneEv={}}` line with:

```js
  // Values were bare ISO strings before ticks were synced; coerce so old ticks survive.
  try{const raw=JSON.parse(localStorage.getItem(LS.doneEv)||"{}");S.doneEv={};
    Object.keys(raw).forEach(k=>{const v=raw[k];S.doneEv[k]=typeof v==="string"?{at:v}:v})}
  catch(e){S.doneEv={}}
```

- [x] **Step 2: Fix the completed-fold sort for the new shape**

In `doneAssignments()`, the sort reads the value as a date directly. Replace it with:

```js
    .sort((x,y)=>new Date(S.doneEv[evKey(y)].at)-new Date(S.doneEv[evKey(x)].at))}
```

`evDone()` only tests truthiness and needs no change — confirm this by reading it.

- [x] **Step 3: Add the tick functions to the sync layer**

Append to the `// ── Google sync ──` section:

```js
// Canvas coursework has no completion flag of its own. A tick becomes a completed Google
// Task carrying a [canvas:calId|eventId] marker, so it crosses devices and shows as done in
// the Tasks app too. pullTasks() diverts anything carrying that marker back into S.doneEv —
// without that, every tick would come back down as a duplicate card in the rail.
function resolveCat(course){
  const c=String(course||"").toLowerCase().replace(/\s+/g," ");
  return S.cats.find(x=>!isExcluded(x)&&c.includes(x.toLowerCase().replace(/\s+/g," ")))
    ||"Coursework"}
async function tickPush(ev){
  const key=evKey(ev),sc=splitCourse(ev.title),cat=resolveCat(sc.course);
  const listId=listMap[cat]||await ensureList(cat);
  const body={title:sc.title||ev.title,status:"completed",
    notes:buildNotes({tag:"General",canvas:key}),
    due:new Date(iso(ev.start)+"T00:00:00Z").toISOString()};
  const d=await tapi(`${listPath(listId)}/tasks`,{method:"POST",body:JSON.stringify(body)});
  S.doneEv[key]={at:new Date().toISOString(),taskId:d.id,listId};saveDone()}
// Takes the record, not the key — the caller removes the entry optimistically before calling.
function tickDel(rec){
  if(!rec||!rec.taskId||!rec.listId)return Promise.resolve();
  return tapi(taskPath(rec.listId,rec.taskId),{method:"DELETE"})}
```

- [x] **Step 4: Divert marked tasks out of the rail and into `doneEv`**

In `pullTasks()`, declare `const remoteTicks={};` next to `const pulled=[];`, then change the `forEach` body from Task 3 Step 5 so the marker is intercepted before anything is pushed:

```js
      (d.items||[]).forEach(it=>{
        const n=parseNotes(it.notes),gdate=it.due?it.due.slice(0,10):null;
        if(n.canvas){ // a coursework tick, not a task — never let it reach the rail
          remoteTicks[n.canvas]={at:it.completed||it.updated,taskId:it.id,listId};return}
        const due=(n.due&&gdate&&n.due.slice(0,10)===gdate)?n.due:(gdate?gdate+"T23:59":null);
        pulled.push({id:it.id,cat,text:it.title||"(untitled)",tag:n.tag,due,
          notes:n.text,canvas:n.canvas,done:it.status==="completed"})});
```

Then, immediately before `S.tasks=[...pulled,...keep];`, reconcile ticks:

```js
    // A tick that reached Google but is no longer there was unticked on another device.
    // Entries with no taskId are local-only (offline tick) and must survive untouched.
    Object.keys(S.doneEv).forEach(k=>{
      const rec=S.doneEv[k];
      if(rec&&rec.taskId&&!remoteTicks[k])delete S.doneEv[k]});
    Object.assign(S.doneEv,remoteTicks);saveDone();
```

- [x] **Step 5: Push the tick from the handler**

Replace the whole `$$("[data-evdone]")` handler with:

```js
  $$("[data-evdone]").forEach(b=>b.onclick=e=>{
    e.stopPropagation(); // the card behind it opens the event editor
    const k=b.dataset.evdone,rec=S.doneEv[k];
    if(rec){
      delete S.doneEv[k];saveDone();render();
      tickDel(rec).catch(err=>{S.doneEv[k]=rec;saveDone();render();
        toast("Sync failed: "+err.message,true)})}
    else{
      S.doneEv[k]={at:new Date().toISOString()};saveDone();render();toast("Completed");
      const ev=S.upcoming.find(x=>evKey(x)===k);
      if(ev)tickPush(ev).catch(()=>{})}}); // stays local; reconciles on the next pull
```

- [ ] **Step 6: Verify the tick crosses devices and does not double up**

1. Tick a Canvas coursework card. Expected: it drops into the Completed fold, exactly once.
2. Check the Tasks rail. Expected: **no** new card appeared there. This is the double-entry check.
3. Check the Google Tasks phone app under that course's list. Expected: the item is there, completed.
4. In profile B, hard-refresh. Expected: the same item shows as ticked.
5. Untick it in profile B, then hard-refresh profile A. Expected: it is back in the Tasks feed.

- [x] **Step 7: Commit**

```bash
git add index.html
git commit -m "Sync Canvas coursework ticks through Google Tasks

A tick becomes a completed task carrying a [canvas:...] marker, which
pullTasks diverts back into S.doneEv so it never doubles as a rail card.
S.doneEv values gain taskId/listId; legacy ISO strings are coerced on load."
```

---

## Task 5: Reminders on a dedicated calendar

The durable path — Google's own infrastructure delivers these, so the phone buzzes with the dashboard closed.

**Files:**
- Modify: `index.html` — the sync layer, `loadCals()` (~line 594), `mapEvents()` (~line 601), plus lifecycle calls in `commitTaskEdit`, `commitDueEdit` and three `wire()` handlers

**Interfaces:**
- Consumes: `api()` and its `.status` property (Task 1), `taskPush` (Task 2), `TZ`.
- Produces: `remId(taskId) -> string`, `wantsRem(t) -> boolean`, `ensureRemCal() -> Promise<string>`, `remindUpsert(t) -> Promise`, `remindDel(t) -> Promise`, and the module-level `remCal` id.

- [x] **Step 1: Add the reminder functions to the sync layer**

Append to the `// ── Google sync ──` section:

```js
// Reminders live on their own calendar so Google delivers them even with the dashboard
// closed. The calendar is hidden from every dashboard surface (see loadCals/mapEvents) —
// otherwise every reminder would surface a second time as an Events card and a grid block.
const REM_CAL_NAME="Dashboard Reminders";
let remCal=localStorage.getItem("bd.remCal")||null;
async function ensureRemCal(){
  if(remCal)return remCal;
  const d=await api("/calendars",{method:"POST",
    body:JSON.stringify({summary:REM_CAL_NAME,timeZone:TZ})});
  remCal=d.id;localStorage.setItem("bd.remCal",remCal);return remCal}
// Calendar requires base32hex event ids (a-v, 0-9), 5-1024 chars. Plain hex is a valid
// subset, so encoding the task id gives a legal, deterministic, collision-free id — which
// is what makes the upsert idempotent without storing a task->event map anywhere.
const remId=tid=>"bd"+[...String(tid)].map(c=>c.charCodeAt(0).toString(16).padStart(2,"0")).join("");
// A dateless ASAP task gets no event: there is nothing to schedule, and a synthetic "9am
// today" would need recreating every morning and would strand stale events behind it.
const wantsRem=t=>!t.done&&!!t.due&&(t.tag==="ASAP"||t.tag==="Deadline");
async function remindUpsert(t){
  if(!wantsRem(t))return remindDel(t);
  const cal=await ensureRemCal(),start=new Date(t.due);
  const body={id:remId(t.id),summary:t.text,description:t.cat,
    start:{dateTime:start.toISOString(),timeZone:TZ},
    end:{dateTime:new Date(start.getTime()+15*60000).toISOString(),timeZone:TZ},
    reminders:{useDefault:false,
      overrides:[{method:"popup",minutes:0},{method:"popup",minutes:60}]}};
  try{await api(`/calendars/${encodeURIComponent(cal)}/events`,
    {method:"POST",body:JSON.stringify(body)})}
  catch(e){
    if(e.status!==409)throw e;   // 409 = already there; anything else is a real failure
    await api(`/calendars/${encodeURIComponent(cal)}/events/${remId(t.id)}`,
      {method:"PUT",body:JSON.stringify(body)})}}
async function remindDel(t){
  if(!remCal)return;
  try{await api(`/calendars/${encodeURIComponent(remCal)}/events/${remId(t.id)}`,
    {method:"DELETE"})}catch(e){}}  // already gone is the desired end state
const remSync=t=>remindUpsert(t).catch(()=>{}); // best-effort at every call site
```

- [x] **Step 2: Verify `remId` produces a legal Calendar id**

Hard-refresh, then in the console:

```js
(()=>{
  const id=remId("aBc-123_xyz");
  console.assert(/^[a-v0-9]{5,1024}$/.test(id),"legal base32hex id, got "+id);
  console.assert(remId("x")===remId("x"),"deterministic");
  console.assert(remId("x")!==remId("y"),"collision-free");
  console.log("remId OK ->",id);
})()
```

Expected: `remId OK -> ...` with no failures.

- [x] **Step 3: Hide the reminders calendar from every surface**

In `loadCals()`, replace the `S.cals=(d.items||[]).map(...)` assignment with a filtered version. Filtering here also stops `loadEvents()` from ever fetching it:

```js
  S.cals=(d.items||[]).filter(c=>{
    const nm=c.summaryOverride||c.summary;
    return c.id!==remCal&&nm!==REM_CAL_NAME})   // name check covers a first run before the id is cached
    .map(c=>({id:c.id,name:c.summaryOverride||c.summary,primary:!!c.primary,
    gcolor:c.backgroundColor||"#8B5CF6"}));
```

And in `mapEvents()`, as belt-and-braces if the calendar is ever renamed, add as the first line of the function body:

```js
  if(calId&&calId===remCal)return[];
```

- [x] **Step 4: Hook the lifecycle**

Add `remSync(t)` after the existing sync call in each of these, so a reminder tracks its task:

- Tag cycle handler (tag may have moved into or out of ASAP/Deadline): `remSync(t);`
- Toggle done handler (a completed task must lose its reminder): `remSync(t);`
- `commitTaskEdit` (the event summary is the task text): `remSync(t);`
- `commitDueEdit` (the event time is the due date): `remSync(t);`

In the remove (`data-rm`) handler, add before `taskDel(...)`:

```js
    remindDel(t).catch(()=>{});
```

In the add (`data-add`) handler, replace `taskPush(t).catch(()=>{});` with:

```js
      taskPush(t).then(()=>remSync(t)).catch(()=>{}); // needs Google's id for remId()
```

Note the ordering: `remId()` is derived from the task id, and `taskPush` rewrites `t.id` from the local `uid()` to Google's. Calling `remSync` first would key the reminder to an id that is about to be discarded.

- [ ] **Step 5: Verify reminders land and stay hidden**

1. Add a task tagged `Deadline` with a due date an hour or two out.
2. Open Google Calendar. Expected: a **Dashboard Reminders** calendar exists with a 15-minute event at the due time.
3. Back in the dashboard, hard-refresh. Expected: that event does **not** appear in the Upcoming Events feed, does **not** appear in the week grid, and **Dashboard Reminders** does **not** appear in the Controls calendar list. This is the duplicate-surface check.
4. Change the task's due date. Expected: the event moves rather than a second one appearing.
5. Tick the task complete. Expected: the event disappears from Google Calendar.

- [x] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add reminders on a dedicated hidden calendar

Deadline and dated ASAP tasks upsert a 15-minute event with popup overrides
at 60 and 0 minutes, so Google delivers the notification with the dashboard
closed. Event ids are hex-encoded task ids, which are legal base32hex and make
the upsert idempotent with no stored map. The calendar is filtered out of
S.cals and mapEvents so reminders never double as Events cards."
```

---

## Task 6: In-page notifications

The same-session convenience on the laptop. On iOS this only fires for an installed PWA (16.4+), which is why Task 5 is the real cross-device path.

**Files:**
- Modify: `index.html` — `LS` map (~line 400), a new function block near `loadWeather()` (~line 657), the `syscol` panel in `render()` (~line 788), `wire()`

**Interfaces:**
- Consumes: `urgentTasks()`, `upcomingAssignments()`, `evKey`, `splitCourse`, `iso` — all existing.
- Produces: `notifCheck()`, run on a 60-second interval.

- [x] **Step 1: Add the notified-key store to `LS`**

Add to the `LS` object literal:

```js
  notified:"bd.notified",
```

- [x] **Step 2: Add the check**

Insert immediately before `function loadWeather(){`:

```js
// Fires for anything falling due inside the next hour. Keys already fired are kept for the
// day so a refresh does not re-buzz; the set resets when the date rolls over. iOS Safari
// only delivers these to an installed PWA (16.4+) — the reminders calendar is the path that
// works with the dashboard closed.
function notifCheck(){
  if(!("Notification"in window)||Notification.permission!=="granted")return;
  let st;
  try{st=JSON.parse(localStorage.getItem(LS.notified)||"{}")}catch(e){st={}}
  const today=iso(new Date());
  if(st.date!==today)st={date:today,keys:[]};
  const fired=new Set(st.keys||[]),now=Date.now(),soon=now+3600000;
  const hit=(k,title,body)=>{if(fired.has(k))return;fired.add(k);
    try{new Notification(title,{body,icon:"icon.svg",tag:k})}catch(e){}};
  urgentTasks().forEach(t=>{if(!t.due)return;
    const d=new Date(t.due).getTime();
    if(d>now&&d<=soon)hit("t:"+t.id,"Due soon · "+t.cat,t.text)});
  upcomingAssignments().forEach(e=>{if(e.allDay)return;
    const d=e.start.getTime();
    if(d>now&&d<=soon)hit("e:"+evKey(e),"Due soon",splitCourse(e.title).title)});
  localStorage.setItem(LS.notified,JSON.stringify({date:today,keys:[...fired]}))}
```

- [x] **Step 3: Add the permission button**

Permission must come from a click — Chrome penalises unprompted requests. In `render()`, inside the `syscol` div, immediately after the `</select>` and before `${readout()}`:

```js
            ${"Notification"in window?`<button class="tbtn" id="notif" ${
              Notification.permission==="granted"?"disabled":""}>${
              Notification.permission==="granted"?"Notifications on"
              :Notification.permission==="denied"?"Notifications blocked"
              :"Enable notifications"}</button>`:""}
```

- [x] **Step 4: Wire the button and start the timer**

In `wire()`, alongside the other `syscol` handlers:

```js
  const nb=$("#notif");
  if(nb)nb.onclick=()=>{Notification.requestPermission().then(()=>{render();notifCheck()})};
```

And at the bottom of the file, next to the existing `setInterval(loadWeather,...)` line:

```js
setInterval(notifCheck,60000);
```

- [ ] **Step 5: Verify**

1. Open Controls, click **Enable notifications**, accept the browser prompt. Expected: the button changes to `Notifications on` and is disabled.
2. Add a `Deadline` task due ~30 minutes out.
3. Within 60 seconds. Expected: a desktop notification titled `Due soon · <category>`.
4. Hard-refresh and wait another 60 seconds. Expected: **no** repeat notification for the same task.

- [x] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add in-page notifications for anything due within the hour

Permission is requested from an explicit Controls button rather than on load.
Fired keys are kept for the day so a refresh does not re-buzz."
```

---

## Task 7: Correct CLAUDE.md

`CLAUDE.md` still describes Google Tasks sync as planned and unbuilt, which is what sent this whole investigation down the wrong path at the start. It is git-ignored, so this is a local edit with no commit.

**Files:**
- Modify: `CLAUDE.md` (git-ignored — **do not** `git add` it)

- [x] **Step 1: Replace the "planned, not yet built" section**

Retitle `## Google Tasks sync — planned, not yet built` to `## Google sync` and rewrite its body to record what now exists:

- Tasks live in Google Tasks, one list per category, mapped in `bd.listMap`.
- Notes carry a machine header — `[tag]`, `[due:...]`, `[canvas:...]` — parsed by `parseNotes()` and written by `buildNotes()`. The due marker exists because Google Tasks discards the time component; **Google wins any date disagreement**, because the Tasks app is the only thing that can change a date behind the dashboard's back.
- Canvas coursework ticks are completed Google Tasks bearing a `[canvas:calId|eventId]` marker. `pullTasks()` diverts them into `S.doneEv` and keeps them out of the rail — removing that check reintroduces duplicate cards.
- `S.doneEv[key]` is `{at, taskId, listId}`, not a bare ISO string. Legacy strings are coerced in `loadLocal()`.
- Reminders live on a **Dashboard Reminders** calendar, hidden in both `loadCals()` and `mapEvents()`. Event ids are hex-encoded task ids so the upsert is idempotent with no stored map.
- **Prefs are deliberately NOT synced.** Categories, colours, calendar visibility, view and collapse state stay per-device. Drive `appDataFolder` was designed and dropped — it was the only piece needing an OAuth scope beyond what is already granted. See the spec's "Explicitly out of scope".

- [x] **Step 2: Update the two now-wrong notes elsewhere in the file**

- Under **Completion**, `S.doneEv` is described as "deliberately not synced anywhere — it is local to whichever device ticked it". That is no longer true; replace it with the Task 4 behaviour.
- Under **Still on the list**, delete the "Google Tasks sync (spec above) — highest priority" bullet. Leave the command-box bullet.

- [x] **Step 3: Add the prerequisite note**

Under **Auth notes**, record that the **Google Tasks API must be enabled** in the Cloud Console, that this was the original silent failure, and that the `CAL·IO` / `TSK·IO` rows in the Controls readout are how to check it at a glance.

- [x] **Step 4: Verify nothing personal is staged**

```bash
git status --short
```

Expected: `CLAUDE.md` does **not** appear. It is git-ignored and the repo is public.

---

## Deploy

```bash
git push origin main
```

GitHub Pages redeploys automatically. Hard-refresh to bypass cache, on the laptop and on the phone.
