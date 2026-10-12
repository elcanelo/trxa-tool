# TRXA-TOOL Studio manual

Studio is a local browser UI for the same pipeline [MANUAL.md](MANUAL.md) walks through via the
CLI: parsing a `.axtr` recording, drafting inputs, editing a Test Case's `outputs[]`/
`expectations[]`, managing Fixtures, building Suites (including their dependency DAG and
conditional-skip gates) and Plans, and reviewing Test Runs -- without hand-editing JSON. It runs
the same code the CLI does, so anything Studio writes is exactly as valid as anything the CLI
writes, and vice versa -- switch between them freely on the same files.

---

## 1. Starting Studio

```sh
trxa studio          # starts the server + client in the background, opens your browser
trxa studio stop     # shuts down the whole process tree (not just the top-level process)
```

Running `studio` again while it's already up just reopens the browser instead of starting a
second instance -- PID/log tracking lives in `~/.trxa-tool/studio-run/` (a per-user location, independent of
where the tool is installed).

**Which project does it open?** Studio's data (`recordings/`, `testcases/`, `suites/`, etc.) lives
under a *working folder* -- a real, persisted setting, not just "wherever you happened to run the
command from." The first time Studio ever starts on a machine, it defaults to and remembers
whatever directory `trxa studio` was run from; after that, it's whatever you last set it to.
Change it anytime from the **Settings** tab (gear icon, top right) -- the switch takes effect
immediately, no restart needed. Install location, working folder, and "wherever you happen to be
running the command from" are three different things, and only the working folder setting decides
which project's files Studio reads and writes.

Studio binds to `127.0.0.1` only and opens at `http://localhost:5173`. **Never expose it on a
network** -- it does unauthenticated filesystem writes by design (same trust model as the CLI:
whoever can run it can write anywhere under the working folder's `testcases/`, `testdata/`,
`testscripts/` directories).

## 2. The six tabs, at a glance

| Tab | Purpose |
|---|---|
| **Recordings** | Parse a `.axtr` file, draft inputs from it, and create a new Test Case + Fixture pair. |
| **Test Cases** | Edit an existing Test Case's `outputs[]`/`expectations[]`, or delete it. |
| **Fixtures** | Full CRUD on `testdata/*.json` Fixtures -- concrete input values for a Test Case. |
| **Suites** | A drag-and-drop dependency-graph canvas for `suites/*.json` -- `dependsOn`, `inputBindings`, and conditional-skip (`runIf`) gates. |
| **Plans** | Compose a `plans/*.json` from existing suites, with reorderable run order and a rollup-status viewer. |
| **Test Runs** | Plan-first, read-only viewer: pick a Plan's run (a real `runs/*.json` PlanRun record), see its suites' rollup, drill into one for per-test-case status, outputs, assertions, live auto-refresh while a run is in progress. |

Every tab's search box, and the active tab + open record itself, round-trip through
`location.hash` -- a refresh, a bookmarked link, or the browser's back/forward buttons all land
back where you were instead of resetting to the Recordings tab. The header also carries **Export
library**/**Import library** buttons that bundle or restore the whole `recordings/`/`testcases/`/
`testdata/`/`testscripts/`/`suites/`/`plans/`/`testruns/` tree as one zip (same as the CLI's
`export`/`import` commands) -- useful for backing up or moving an installation.

Not here yet: creating a Test Case's `inputs[]`/`bindTo` after the fact -- that stays on the
Recordings page's flow below.

## 3. Recordings: parse a `.axtr` file and draft a Test Case + Fixture

The left pane lists every `.axtr` file under `recordings/`, grouped into a folder tree matching
their layout on disk (search flattens it to a filtered list with full paths, since matches from
different folders no longer need visual grouping once you're searching). Click one to parse it.

Once parsed, you'll see the recording's name, step count, and required-privilege count, followed
by a step table: sequence, description, form, node type, and the suggested replay hint for that step
(the tool it'd call during replay, a confidence badge, and a note when the mapping needed one).
Steps with no hint show an "unmapped" badge -- rare, but worth reporting if you see one.

Click **Suggest Inputs** to draft candidate inputs from the recording (same output as
`trxa suggest-inputs`). Each candidate traces back to either a recorded value
(`PropertyUserAction`) or a low-confidence grid-selection step, and renders as an editable row
directly under the step it came from -- required candidates are marked with a red `*`. Edit the
values here before creating the Test Case; whatever you type becomes that draft input's initial
value in the Fixture you create alongside it.

**Caution:** candidate values are copied verbatim from the recording -- if it was made against
real business data rather than demo data, review/scrub these before creating anything you intend
to commit. See MANUAL.md's "Data hygiene" section.

Once you've reviewed the candidates, fill in the **Create Test Case & Fixture** form:

- **Test case ID** -- written to `testcases/<id>.json` and `testscripts/<id>.testscript.json`.
- **Test case name** -- pre-filled from the recording's own name; edit freely.
- **Description** -- optional.
- **Fixture name** -- type a new one or pick from the autocomplete of existing fixture names
  (e.g. `basic`). Written to `testdata/<id>.<name>.json`.

**Create Test Case & Fixture** stays disabled until every required candidate has a value and all
four fields above are filled. On success it writes all three files in one action; if the Test
Case is created but the Fixture write fails for some reason, the error banner says so explicitly
-- you can still create the Fixture separately from the Fixtures tab using the same Test Case ID.

## 4. Test Cases: edit outputs and expectations

The left pane groups every `testcases/*.json` file into a folder tree matching its source
recording's location under `recordings/` (an entry that fails to parse/validate shows "Invalid:
<error>" instead of crashing the list; one created before this grouping existed, or from a
top-level recording, falls into an "Ungrouped" bucket). Click one to load it, along with its
underlying `testscript.json` (used to populate the step pickers below).

If the Test Case has `inputs[]`, they're shown read-only at the top with a reminder of the
`{{paramName}}` substitution syntax you can use in a filter/value below (e.g.
`CustomerAccount eq '{{customerAccount}}'`) -- editing inputs themselves stays on the Recordings
page's flow (§3).

### Outputs

Each output captures a value after some step, for a later Test Case in a suite to consume (see
`{ "kind": "testCaseOutput", ... }` bindings in MANUAL.md §7). The **name** field is free text --
whatever you call it here is what a downstream Test Case's binding refers back to.

Pick a **kind**:

- **`odata`** -- query the entity via OData: `Entity set`, `Filter` (may use
  `{{paramName}}`), `Field` to read. Preferred when there's a solid, unique business key to
  filter by -- durable, works even after the session ends.
- **`uiField` (form state)** -- read a value straight off the live form. Pick the step to read it
  **after**; then either:
  - **Control name** -- the exact AOT control name. Free at replay time (no extra query, the
    value is already in that step's own response) and exactly reproducible. Auto-suggested from
    the selected step when it acts on one control; otherwise pick from the autocomplete of every
    control name seen anywhere in this script, or type your own.
  - **Hint** (only used when Control name is blank) -- a plain-English description (e.g. "the
    newly assigned purchase order number shown after clicking New"), for values Task Recorder
    never captured because nobody clicked them -- typically an auto-assigned ID. The replaying
    LLM resolves this against live form state during a run and is expected to write the resolved
    control name back into this field for you, so it stays free/exact on every run after the
    first.

### Expectations

Each expectation checks something must (or must not) hold after a step, for the Test Case to
pass. Pick the step to check **after**, then a **kind**:

- **`odata`** -- query an entity and compare `Expected field` against `Expected value` with a
  `Comparator` (`equals`/`notEquals`/`contains`/`greaterThan`/`lessThan`).
- **`uiMessage`** -- check `FormState.Messages` for an entry of the given **Severity**
  (`Info`/`Warning`/`Error`) whose text contains **Pattern** as a plain substring, not a regex --
  leave it blank to match any message of that severity regardless of text.

**Watch the `Should exist` checkbox carefully** -- it's easy to get backwards. If your
expectation reads "no error appeared," you want `Severity: Error` with **`Should exist`
unchecked**, not checked -- checking it asserts an Error-severity message *must* appear, the
opposite of what "no error" usually means. (This exact mistake has shipped in a real test case before
being caught during a live replay -- worth double-checking every time.)

### Saving and deleting

**Save** writes the file back (validated server-side the same way the CLI's
`validate-testcase` command would reject a malformed one) and confirms with a toast. **Delete**
asks for confirmation, then removes only `testcases/<id>.json` -- not the `testscript.json` it
points to, since that file isn't guaranteed unique to one Test Case (a suite's test cases can
share a source recording), and not any Fixture/Suite that still references this Test Case's ID
(those will start failing to resolve; clean them up separately).

## 5. Fixtures: manage test data

The left pane groups every `testdata/*.json` file by the Test Case it belongs to (same
"Invalid: ..." handling as Test Cases), so a scenario with several data variants (e.g. three
vendors × three items) stays organized instead of one long flat list. **+ New fixture** switches
the detail pane to create mode.

For a **new** fixture: pick a **Filename slug** (written to `testdata/<slug>.json` -- convention
is `<testCaseId>.<name>`, e.g. `create-purchase-order.basic`, but nothing enforces it), a **Test
case** from the dropdown, and an optional **Label** -- a short free-text description (distinct
from the filename-safe slug) shown in the fixture's list entry so multiple variants for the same
Test Case stay distinguishable at a glance. Once you pick a Test Case, the **Values** section
below renders one field per its `inputs[]` -- label, required marker, placeholder from the
input's recorded `example`, and its description as a hint underneath.

**The Test case a Fixture targets can't be changed after creation** -- the field is locked on an
existing Fixture; delete and recreate it against a different Test Case instead. This is
deliberate: a Fixture's Values shape is entirely derived from its Test Case's `inputs[]`, so
repointing it would silently invalidate every value already entered.

**Save**/**Delete** work the same as Test Cases (§4) -- Save confirms via toast, Delete asks for
confirmation and removes only that one `testdata/*.json` file.

## 6. Suites: the dependency graph canvas

The left pane lists every `suites/*.json` file. **+ New suite** switches the detail pane to
create mode, asking for a **Filename slug** (written to `suites/<slug>.json`), a **Suite id** (the
stable id a Plan references it by -- independent of the filename), and a **Name**.

The main panel is a top-to-bottom flow canvas (`@xyflow/react`): one node per entry, arrows
showing `dependsOn`. **Drag from a node's bottom handle to another node's top handle** to add a
dependency; select an edge and press **Backspace or Delete** to remove one. Drag a node itself to
reposition it -- positions are presentation-only (kept in your browser's `localStorage`, not
written to the suite file), and a **Reset layout** button (shown once you've moved something)
clears them back to the automatic layered layout. Use the toolbar's dropdown + **Add** to add an
existing Test Case as a new entry.

Click a node to open its inspector:

- **Legal entity** -- passed as the company on this entry's Test Case's *first* navigation call
  only; leave blank to stay in whatever company the run is already in.
- **Depends on** -- read-only list of what you've connected via the canvas.
- **Run condition** -- an optional conditional-skip gate. Pick one of this entry's own
  dependencies from the **Only run if** dropdown (or "Always run" to clear the condition), then
  an **output** that dependency declares, a **comparator** (`equals`/`notEquals`/`contains`/
  `greaterThan`/`lessThan`), and an **expected value**. If the condition evaluates false at replay
  time, this entry is recorded `"skipped"` instead of running -- not a pass, not a failure. A
  downstream entry that depends on a skipped one is itself reported `"blocked"`, so nothing is
  ever left waiting forever. Only shown for entries that have at least one dependency, since a
  root entry has nothing to gate on.
- **Input bindings** -- one row per the Test Case's declared `inputs[]`, with a five-way picker:
  **Unbound**, **Fixture** (pick one of the Test Case's fixtures, then a value key from it),
  **Output** (pick an upstream Test Case -- any transitive dependency, not just a direct one --
  and one of its declared outputs), **Literal** (type a value directly into the suite file), or
  **Auto** (Autogenerated -- no value here at all; the orchestrating LLM invents one at replay
  time using the input's own description/example as guidance, typically for a business key that
  just needs to be unique per run).

**Save**/**Delete** work the same as other tabs -- Save validates server-side (including the same
DAG cycle/unknown-id check `validate-suite` runs on the CLI), Delete asks for confirmation and
removes only that one `suites/*.json` file.

## 7. Plans: group suites for combined reporting

The left pane lists every `plans/*.json` file. **+ New plan** switches to create mode, asking for
a **Filename slug**, **Plan id**, **Name**, and **Description**.

The editor is a two-pane picker: **Available suites** (with its own search) on the left, each
with a **+ Add** button; **In this plan (run order)** on the right, each entry with move-earlier/
move-later/remove controls -- array order is the suggested run order, there's no dependency graph
between suites (if two suites need to share data, they belong in one suite instead).

Below the editor, a **rollup-status viewer** shows a combined pass/fail verdict across every suite
in the plan for one PlanRun. **Start new run** creates a real, persisted `runs/<id>.json` record
(`{id, planId, startedAt}`) with an auto-generated id; the **Run** dropdown lists every PlanRun
that belongs to this plan, newest first, by its actual start time. Run
each of the plan's suites (by hand or via the replay skill) using that PlanRun's id as the
`<runId>` in `testruns/<suiteId>-<runId>.json`, exactly as the CLI's `init-plan-run` +
`init-run` sequence expects (see MANUAL.md §8).

For the selected run, each suite gets one of four statuses: **missing** (no
`testruns/<suiteId>-<runId>.json` exists at all -- that suite hasn't been run yet under this
PlanRun), **incomplete** (the file exists, but some test case in it hasn't reached a terminal
status yet), **failed** (every test case reached a terminal status, and at least one is
`failed`/`blocked`), or **passed** (every test case terminal, none failed or blocked). The plan's
own **Overall** badge then picks the worst case across all its suites, in that same order: any
suite `failed` → **has-failures**; else any `incomplete` → **incomplete**; else any `missing` →
**missing-runs**; else → **all-passed**. This is the same logic the CLI's
`plan-status` command uses -- a pure, read-only computation over
already-existing Test Run files, safe to check as often as you like while suites are still
running, since nothing beyond each suite's own Test Run is ever written.

**Save**/**Delete** work the same as Suites.

## 8. Test Runs: view results

The left pane groups by Plan, not by suite file: each Plan you've created is a header, and under
it every PlanRun that belongs to it (see §7) is a clickable row showing when it started, newest
first -- a search box filters by plan name. Selecting a run shows the plan-level rollup (the same
Overall badge plus per-suite status/counts §7 describes), and clicking one of its suites drills
into that suite's own detail below.

That suite detail shows every test case in it: its status badge (`Passed`/`Failed`/`Blocked`/
`Skipped`/`Running`/`Pending`), any captured `outputs[]` values, and every `expectations[]`
assertion with a Pass/Fail badge and the `actual` value that was actually observed (an OData
field, or the matched/missing UI message) -- the same evidence `replay-trxa-testplan` records
when it calls `record-result`. A "failures only" toggle filters the per-test-case list down to
just what needs attention. **While a run is still in progress, this view auto-refreshes itself
every few seconds** -- no manual re-selecting needed to watch it happen live.

If any test case reports `tokensUsed` and/or `costUsd` (see MANUAL.md §7.8's "Token/cost
reporting"), a **Tokens**/**Cost** tile appears alongside the Passed/Failed/Blocked/Skipped/Total
summary row, showing the sum across the whole run; the same figures also show per-test-case in
each result's meta line. Older runs recorded before this existed simply don't show these tiles --
there's no "$0.00" placeholder for a run that never reported either.

**Copy summary** (top-right of an open suite) copies that suite's results to your clipboard as
both Markdown (for pasting into chat) and rich HTML (for pasting into email/Word/Docs, with status
colors and Infolog severity preserved). **Export** offers the same content as a downloadable
`.md` file, or **PDF** via the browser's own print dialog. **Delete** (same row) removes that
suite's Test Run file entirely after a confirmation prompt -- unlike deleting a Test Case/
Fixture/Suite/Plan, a Test Run has no other file that references it by id, so this can't break
anything else. The plan-level rollup view has its own **Delete run**, which only removes the
`runs/<id>.json` PlanRun record itself -- the underlying suite Test Run files aren't touched, so
deleting a PlanRun just un-groups suites that already ran, it doesn't erase their results.

This tab is otherwise **read-only** -- there's no editing a run's contents, since it's a durable
record of what a replay session actually did, not something to hand-edit. To produce a new one,
start it from the Plans tab's rollup viewer (§7), then run each suite live per MANUAL.md §8's
"Running a plan" procedure (itself the same per-suite `resolve-next`/`record-result` loop from
§7.7, once per suite); this tab picks it up on its own (the list itself also polls periodically,
so a run started from another session or the CLI appears without a manual refresh).

## 9. Cross-cutting UI notes

- **Toasts**: every Save on an editable tab (Recordings/Test Cases/Fixtures/Suites/Plans) confirms
  with a small message that fades in at the top of the screen, holds briefly, then fades out on
  its own -- no need to dismiss it. Test Runs has nothing to save, so it never shows one.
- **Error banners**: validation failures (missing required fields, a malformed filter, a suite DAG
  cycle, etc.) show as a banner at the top of the detail pane, not a toast -- toasts are for
  success, banners are for something you need to fix.
- **List/detail split pane**: every tab shares the same layout -- a searchable/scrollable list on
  the left, the selected item's editor (or, for Test Runs, read-only view) on the right. Nothing
  you do in an editable tab's editor is saved until you click Save.
- **URL/refresh persistence**: the active tab and whichever record you have open are reflected in
  the browser's address bar (`location.hash`), so refreshing the page, following a bookmarked
  link, or using back/forward all restore the same screen instead of resetting to Recordings.

## 10. Troubleshooting

- **Port 5173 (or 4321) already in use**: something else is already running there -- check
  `~/.trxa-tool/studio-run/` for a stale PID, or run `trxa studio stop` first. If neither
  clears it, something outside Studio's own tracking is holding the port; find and stop it
  manually (Task Manager on Windows, `lsof -i :5173` on macOS/Linux).
- **A recording/Test Case/Fixture/Suite/Plan doesn't show up**: those lists load once, when you
  open the tab -- refresh if you added a file outside Studio (e.g. via the CLI or by hand) while
  it was open. Test Runs is the exception: both its list and an open run poll periodically on
  their own, so a run started elsewhere (another session, the CLI) appears without a manual
  refresh.
- **"Invalid: ..." in a list**: that file failed schema validation -- the error message is the
  same one `validate-testcase`/`validate-fixture`/`validate-suite`/`validate-plan` would give you
  on the CLI. Fix it by hand or via the CLI; Studio can't open an invalid file for editing.
- **Can't delete a dependency connection on the Suites canvas**: click the connection to select
  it, then press **Backspace or Delete**.
