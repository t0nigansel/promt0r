# UIToDo.md — Making the hamm0r flow intuitive

A step-by-step implementation plan for the UX review
(see the review board and `docs/productVision.md`). Work top to bottom:
each phase builds on the last, and every step ends with a **Verify** line —
a visible result you can confirm before ticking the box.

When every box here is checked, the app tells the truth about its own
mental model, has no dead controls, surfaces findings fast, and is fully
keyboard-drivable.

## How to use this file

- Do steps in order. Phases are sequenced so nothing you fix gets
  contradicted by something you fix later.
- Tick `- [ ]` → `- [x]` only after the **Verify** result is true.
- Keep diffs small: one step = one commit. Commit message states
  What / Why / How tested (per `CLAUDE.md`).
- After every edit the PostToolUse hook runs
  `cargo fmt && cargo clippy --all-targets -- -D warnings && cargo check`.
  Keep it green. UI-only steps also need a manual check in the running app.

## How to run and verify

```
cd crates/hamm0r && cargo tauri dev
```

Most steps are verified by hand in that dev window. Line numbers below are
pointers at time of writing — if one has drifted, the **Verify** step still
describes the observable behaviour, so trust the behaviour over the number.

## Guardrails (do not break these while implementing)

- Core must work with **no analyzer** activated (default install).
- **No cloud call** in the core flow; runner hits only localhost / the
  user's target.
- **Click beats config** — never route the user toward editing a YAML file.
- **One screen, one job** — building an object ≠ running it.
- Do **not** touch the run-JSONL handoff schema (`docs/Datamodel.md`).
  Everything here is UI + export formatting only.

## Progress

- Phase 1 — Truth in copy & dead controls: **7 / 7 ✓**
- Phase 2 — Consistent scoping (the root fix): **2 / 2 ✓**
- Phase 3 — First-run clarity: **6 / 6 ✓**
- Phase 4 — The findings payoff: **2 / 2 ✓**
- Phase 5 — The pentester loop: **6 / 6 ✓**
- Phase 6 — Editor density: **1 / 1 ✓**
- **Total: 24 / 24 ✓**

---

## Phase 0 — Capture the "before" (so improvement is measurable)

- [ ] **Record the cold first-run.** Launch a clean profile
  (`~/hamm0r` empty or renamed), open the app, and try to get from
  zero to a fired run. Note every place you hit a wall, a confusing
  label, or a dead button.
  **Verify:** you have a short written before-list. You will re-run this
  exact walkthrough in the final acceptance step and compare.

---

## Phase 1 — Truth in copy & dead controls

Pure copy and removal changes. No logic risk, immediate visible wins.
These are safe to do first because they don't depend on the scoping
refactor in Phase 2.

- [x] **1.1 — Delete the false "open an Engagement first" copy.**
  Remove the "Make sure an Engagement is open first — use the + button in
  the toolbar" sentence from the Requests welcome
  (`ui/index.html` ~181) and the equivalent "open or create one before
  building Requests or Scenarios" claim in the Engagements welcome
  (`ui/index.html` ~719). Requests are global; there is no topbar "+".
  **Verify:** open Requests with no engagement — the welcome text no
  longer mentions engagements or a toolbar "+", and "+ Create request"
  still works.

- [x] **1.2 — Fix the Quick Start "in the Library" wording.**
  In the Quick Start intro (`ui/index.html` ~984) change "a Request you've
  already configured in the Library" to "…in the Requests view".
  **Verify:** open Quick Start → the intro points users to Requests, not
  Library.

- [x] **1.3 — Stop advertising `config.yaml` in Settings.**
  Remove the "Configuration lives in ~/hamm0r/config.yaml" /
  "…saved to ~/hamm0r/config.yaml" inline notes (`ui/index.html`
  ~1136, ~1155, ~1191, ~1338, and the token modal ~1081). Replace with
  behaviour-focused copy ("Saved automatically" / "Applied after
  restart") or nothing.
  **Verify:** every Settings pane is free of file-path references; saving
  still persists (reopen Settings, values stuck).

- [x] **1.4 — Neutralise the non-functional Import CSV button.**
  `import_csv` throws unconditionally (`ui/js/api.js:779`). Either hide
  the button (`ui/index.html` ~461) or disable it with a "coming soon"
  tooltip and do **not** open the file picker (`ui/js/app.js` ~2381).
  **Verify:** clicking (or hovering) Import CSV never opens a file
  dialog and never produces the "future milestone" error toast.

- [x] **1.5 — Remove the inert "Tester name" field.**
  The Scenario builder field (`ui/index.html` ~600) is never read on save
  and is dropped by `start_scenario`. Delete the field and its label. (If
  you'd rather keep attribution, instead wire it through
  save + `start_scenario` end-to-end — but do not leave it inert.)
  **Verify:** the Scenario builder no longer shows "Tester name", and
  saving + firing a scenario still works.

- [x] **1.6 — Clear the stale topbar progress bar.**
  `clearTopbarProgress()` (`ui/js/app.js:1289`) is defined but never
  called. Call it a few seconds after a run reaches a terminal state
  (in the `API.onProgress` handler when `ev.finished`, ~3707).
  **Verify:** fire a short run; after it finishes the `[====] N/N` bar
  disappears from the topbar instead of lingering green.

- [x] **1.7 — Make error toasts readable and dismissible.**
  In `toast()` (`ui/js/app.js` ~243) give `type === 'error'` toasts a
  manual × dismiss and a longer/no auto-timeout (keep success/info at
  4 s). Allow text selection.
  **Verify:** trigger an error (e.g. fire a request at a bad URL) — the
  error toast stays until dismissed, its text is selectable/copyable.

---

## Phase 2 — Consistent scoping (the root fix)

This is the structural change the whole "engagement-first" confusion
grows from. After this, Requests / Scenarios / Library all behave as the
global objects they already are on disk; an engagement is required only to
**fire**.

- [x] **2.1 — Ungate Scenario building.**
  Remove the `if (!dbOpen) return;` guard in `loadScenarioList()`
  (`ui/js/app.js` ~2410) and the "Open an engagement first" abort in
  `createNewScenario()` (~2466). Running still goes through
  `start_scenario`, which already requires an engagement — that is the
  correct place to gate.
  **Verify:** with no engagement open, go to Scenarios → the list loads,
  "+ Create scenario" opens the builder, and you can save a scenario.
  Firing it still prompts for / requires an engagement.

- [x] **2.2 — Make the Library uniformly global.**
  Replace the "open an engagement to load prompts" placeholder
  (`ui/index.html` ~471) with a neutral "loading prompts…", and drop the
  `if (!dbOpen)` guard on Seed (`ui/js/app.js` ~2397) so Seed matches
  Add/Edit/Delete (already ungated).
  **Verify:** open Library with no engagement → prompts list populates,
  and Seed works without an error toast.

---

## Phase 3 — First-run clarity

Now that scoping is honest, make the on-ramp obvious.

- [x] **3.1 — Keep Quick Start reachable at all times.**
  Quick Start's tile only renders in the `!dbOpen` branch of
  `updateHomeCtas()` (`ui/js/app.js` ~1129). Always include a Quick Start
  tile (or a persistent Home-header action) regardless of `dbOpen`; its
  orchestration creates its own engagement, so it is safe when one is
  already open.
  **Verify:** open any engagement, return Home → a Quick Start entry is
  still present and launches the modal.

- [x] **3.2 — Make the Home flow-strip honest and clickable.**
  The strip (`ui/index.html` ~117) teaches "1 Engagement → 2 Requests →
  3 Scenarios → 4 Run": it isn't clickable, "Run" is no sidebar noun, and
  it omits Library. Rework it to the four real nouns
  (Engagement · Request · Scenario · Library) with Run shown as an action
  inside an Engagement, and make each step route to its view.
  **Verify:** clicking each flow step navigates to the matching view;
  the strip no longer implies a mandatory engagement-first order.

- [x] **3.3 — Add a persistent engagement-state chip in the topbar.**
  Add a small chip that reads "No engagement open — open one" (opens the
  engagement dialog) when none is active, and shows the open engagement's
  name otherwise. It's context, so it belongs in the top bar next to the
  breadcrumb (`ui/index.html` ~17–38).
  **Verify:** with no engagement open the chip is visible and opens the
  dialog; after opening an engagement it shows that engagement's name.

- [x] **3.4 — Show the sidebar nouns as words, and regroup Library.**
  Sidebar labels are `opacity:0` until hover (`ui/style.css` ~654/673).
  Render icon + label persistently (a wider labelled rail, or at least
  until the first completed run). Move Library above the `nav-spacer`
  (`ui/index.html` ~88) into the noun group; leave only Settings (and
  optionally Home) below it.
  **Verify:** all six nav items show their text without hovering; Library
  sits with Engagements/Requests/Scenarios, not next to Settings.

- [x] **3.5 — Explain the empty Verdict column in core-only mode.**
  When no analyzer is active, `engagementVerdictBadgeHtml` renders "—"
  for every row (`ui/js/app.js` ~3287). Either hide the Verdict column
  or give the "—" a tooltip/inline note: "Verdicts require Analyz0r —
  activate in Settings" (reuse `analyzerUnavailableReason()`).
  **Verify:** on a fresh core-only install, a completed run's Verdict
  column is either absent or clearly explains how to enable scoring —
  it never looks broken.

- [x] **3.6 — Auto-derive the Request Id from its Name.**
  Slugify the Id from the Name on save (mirror the Prompt editor, which
  already says "id is auto-derived from the name"). Drop the
  "filename stem, kebab-case" label (`ui/index.html` ~193). Keep an
  optional manual override under the Advanced disclosure added in
  Phase 6 (step 6.1).
  **Verify:** create a Request typing only a Name → it saves with a
  sensible auto id, no "Id" field required, no "filename stem" jargon.

---

## Phase 4 — The findings payoff

The report is the deliverable. Make findings easy to reach and complete.

- [x] **4.1 — Put Verdict + Triage into the PDF and Markdown exports.**
  Add Verdict and Triage/Note columns to `buildRunExportHtml()`
  (`ui/js/app.js` ~4637) and a Verdict column to `buildMarkdownReport()`
  (~4490). Pull triage from `engagementDetail.triageByRunId` and verdict
  from `r.judge_verdict` — both already on the rows. (Export formatting
  only; the run-JSONL schema is untouched.)
  **Verify:** triage a couple of results, export PDF and Markdown → both
  show the Verdict and the triage disposition/notes.

- [x] **4.2 — Filter and sort results by verdict (success-first).**
  Add a verdict filter row (Success / Fail / Partial / Unclear / Any)
  beside the triage chips (`ui/index.html` ~795), reusing the
  `row.dataset` + `applyTriageFilter` pattern. Make the Verdict/Status
  headers click-to-sort and default the results view to success-first.
  **Verify:** after a run you can click "Success" to see only cracked
  attempts, and clicking the Verdict header reorders the table.

---

## Phase 5 — The pentester loop (power users)

Make the daily, repeated actions fast — without adding config surfaces.

- [x] **5.1 — Cmd/Ctrl+Enter fires the active view's object.**
  `resolveGlobalFireAction()` (`ui/js/app.js` ~1234) already computes the
  right per-view action, but its trigger `#btn-global-fire` does not exist
  in the DOM, so the path is dead. In the existing keydown handler
  (~5163) bind Cmd/Ctrl+Enter to that logic and dispatch to
  `fireSelectedRequest` / scenario / engagement based on `activeViewId`
  (ignore when focus is in an input/textarea). Delete the orphaned
  `#btn-global-fire` handler (~3478) and trim `updateGlobalFireButton`.
  **Verify:** on Requests/Scenarios/Engagement detail, Cmd/Ctrl+Enter
  fires the selected object; typing in a text field does not; no dead
  `#btn-global-fire` references remain.

- [x] **5.2 — One-click replay from a results row + replay history.**
  Add a row-level replay icon in the results table (mirror the runs-table
  rerun icon ~4185) that calls `replay_attempt` directly for an instant
  re-fire. In the Replay panel, render a history list from `list_replays`
  (`ui/js/api.js` ~513) — currently never called — so variations can be
  compared, and let a promising replay be triaged.
  **Verify:** re-fire an attempt from a results row in one click; open a
  result's detail and see prior replays listed, each viewable.

- [x] **5.3 — Wire up the curl-import modal.**
  The "Import curl" modal exists in markup (`ui/index.html` ~927) with
  **no handlers**. Add an "Import curl" button in the Request editor
  actions, open the modal, and parse the pasted command into
  `populateRequestEditor()` shape (URL, method, headers, body, auth).
  **Verify:** paste a real `curl` command → the Request form fills in
  correctly and saves. (If parsing is deferred, delete the dead modal
  instead so it can't be mistaken for a live feature.)

- [x] **5.4 — Bulk-apply triage across filtered rows.**
  Add a "set status for all filtered rows" control to the triage filter
  bar (`ui/js/app.js` ~3341/3352) that iterates `set_triage_status` over
  the currently visible seqs; optionally support shift-click range
  select.
  **Verify:** filter to a set of rows, choose "mark all False Positive" →
  every visible row updates in one action.

- [x] **5.5 — Speed up the daily re-run loop.**
  Home "Resume" only opens an engagement (`ui/js/app.js` ~740).
  `runPickedScenario` always creates a fresh `"<scenario> · run"`
  engagement (~806). Add a "▶ Re-run" action on Home recent rows that
  fires the engagement's bound scenario back into that same engagement;
  let "Run a Scenario" optionally target an existing engagement; and give
  auto-created engagement names a timestamp so repeats stay
  distinguishable.
  **Verify:** re-running yesterday's scenario is one click and lands as a
  new run inside the existing engagement (no duplicate identically-named
  workspaces).

- [x] **5.6 — Keyboard view-switching + a live-run indicator that
  navigates.**
  Add Cmd/Ctrl+1–5 (or a `g`-chord) in the keydown handler (~5163) to
  call `showView()`. Track the engagement slug alongside `run_id` in the
  topbar progress state (~1300) and make the topbar bar / breadcrumb
  segments jump to that run's results.
  **Verify:** number shortcuts switch views; clicking the active-run
  topbar indicator jumps straight to that run's results; breadcrumb
  segments navigate up. (Accelerators only — no command-palette /
  config surface, per the vision.)

---

## Phase 6 — Editor density

- [x] **6.1 — Collapse advanced fields behind disclosures.**
  In the Request editor, group Tag, the "Bind extracted value as"
  auth-chain field with its long hint, and "Result columns"
  (`ui/index.html` ~197, ~283, ~299) under one collapsed "Advanced"
  section. In the Scenario builder, wrap Mutations (~646) and
  Multi-session (~661) in sections collapsed by default when inactive
  (no mutators enabled / session count = 1), each showing a one-line
  summary.
  **Verify:** a first-contact Request editor shows only the essentials
  (Name, Method+URL, Headers, Auth, Body, extract, Timeout, Test) with
  Save/Fire visible without scrolling; advanced fields are one click
  away. The Scenario builder shows Mutations/Multi-session collapsed
  until enabled.

---

## Acceptance — confirm the UI is actually better

- [ ] **A.1 — Re-run the Phase 0 first-run walkthrough** on a clean
  profile. Every wall from your before-list is gone: no false
  "engagement first" instruction, no dead/erroring controls, Quick Start
  always reachable, findings filterable, exports carry verdicts.
  **Verify:** a first-timer reaches a fired run without hitting a false
  gate; your before/after notes show the removed friction.

- [ ] **A.2 — Power-user loop check.** Fire, triage in bulk, replay from a
  row, re-run from Home, and switch views — all without touching the
  mouse where a shortcut now exists.
  **Verify:** the repeated daily actions each take fewer clicks than
  before, with keyboard paths available.

- [x] **A.3 — Guardrails intact.** `cargo clippy --workspace
  --all-targets -- -D warnings` is clean; core still runs with no
  analyzer; no new cloud calls; the run-JSONL schema is unchanged.
  **Verify:** clippy green, a core-only run completes and exports raw
  data.
