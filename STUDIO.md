# TRXA Studio manual

Studio is a local browser UI for the same pipeline [MANUAL.md](MANUAL.md) walks through via the
CLI: parsing a `.axtr` recording, drafting inputs, editing a Test Case's `outputs[]`/
`expectations[]`, managing Fixtures, building Suites (including their dependency DAG and
conditional-skip gates) and Plans, and reviewing Test Runs -- without hand-editing JSON. It runs
the same code the CLI does, so anything Studio writes is exactly as valid as anything the CLI
writes, and vice versa -- switch between them freely on the same files.

---

## 1. Starting Studio

Studio runs from the CLI in your browser, or as the TRXA Studio Windows app (see README.md's
"Install"). The app starts its own Studio server on a free local port and shows it in
a window; everything below applies to both, except the `trxa studio` commands, the PID tracking
and the fixed address.

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
| **Fixtures** | Named data sets (`testdata/<name>.json`), each holding input values for many Test Cases. Create, duplicate, rename, delete. |
| **Suites** | A drag-and-drop dependency-graph canvas for `suites/*.json` -- `dependsOn`, `inputBindings`, and conditional-skip (`runIf`) gates. |
| **Plans** | Compose a `plans/*.json` from existing suites, with reorderable run order and a rollup-status viewer. |
| **Test Runs** | A table of everything that was run (plan, suite or test case runs), with filters; open a run for its own page: status, progress, results per suite and test case, and the agent's activity. Live while a run is in progress. |

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
- **Fixture name** -- pick an existing fixture to add this test case's values to it, or type a
  new name to create one (`testdata/<name>.json`).

**Create Test Case & Fixture** stays disabled until every required candidate has a value and all
four fields above are filled. On success it writes the test case and test script and adds the
values to the fixture in one action; if the Test Case is created but saving its fixture values
fails, the error banner says so explicitly -- you can still add the test case to a fixture from
the Fixtures tab.

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

A fixture is one named data set (one company, one scenario) holding values for every test case it
covers, stored as `testdata/<name>.json`. The left pane lists the fixtures themselves, sorted by
name; files that can't be read go in an **Invalid** folder. The search matches fixture names,
descriptions, and the names or ids of the test cases inside them. **+ New fixture** asks only for
a **Name** (what suites refer to, also the file name) and an optional **Description**; the new
fixture then opens so you can add test cases to it.

Selecting a fixture shows its **General** section (name and description) and a **Test cases**
table: one row per test case the fixture has values for, with how many values are set and a
status (**Complete**, **N required missing**, or **Test case not found** when the test case was
deleted). The table has its own search and pages once there are more than ten rows.

- **Add test cases** opens a searchable list of the test cases that have no values in this fixture
  yet; pick any number.
- **Expand a row** to edit that test case's values: one small field per declared input, three to a
  row, labelled with the control's readable label (the input name and description are in the
  info toggletip). Required inputs are checked on blur and again on Save. Values the test case no
  longer declares are kept and listed with a **Remove unused values** action. The header's expand
  toggle opens or closes every row on the page.
- **Remove from fixture** (the row's minus icon) asks for confirmation and names any suite that
  binds this fixture for that test case.

**Save** checks every row and opens the ones with missing required values. Renaming the fixture
in General also updates every suite binding that refers to the old name, and the toast says how
many suites changed. **Duplicate** copies the saved fixture, every test case and value included,
under a new name you choose, then opens the copy; unsaved edits are not copied. **Delete** asks for
confirmation, names the suites that use the fixture, and removes its file.

The **Fixtures** tab on a test case lists the fixtures that hold values for it, each linking here.

**Older fixture files.** Fixtures stored in the earlier one-file-per-test-case format show as
"(old format)" and open read-only, under a notice with a **Merge fixture files** action. It does
the same as `trxa migrate-fixtures` (see MANUAL.md): one file per fixture, old files removed,
nothing changed if two files conflict.

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

Point at a node (or Tab to it) to show its **Edit** and **×** icons in the top-right corner.
**Edit** opens its inspector and **×** removes the entry; clicking or dragging the node itself
only moves it:

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
  **Unbound**, **Fixture** (pick a fixture that has values for this Test Case, then a value key from it),
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

**One row per run.** Whatever was started is one line: a plan run contains its suites and their
test cases, a suite run contains its test cases, and a test case run is just that case. A suite
run that belongs to a plan run is not listed on its own, and an agent run is not a second row
either: it is folded into the run it created (its environment, company, depth and agent show on
that row and on the run's page).

**The table** shows each run's status, name, type (Plan, Suite, Test case), when it started, how
long it took, its results ("7 passed, 1 failed"), the environment and company, and who ran it
(GitHub Copilot, Claude Code, or the CLI). Search matches the name, the plan/suite/test case and
the environment. Filters across the top narrow it by Type, Status, Started (today, last 7 or 30
days), Run by and Environment; **Clear filters** resets them. Columns sort by clicking their
header. Running runs always stay at the top, with a spinner and a progress bar. The table keeps
its filters, sort and page while you open a run and come back.

**Run names.** The Run dialog asks for a name, starting from the plan, suite or test case name so
you can add what makes this run different ("Accounts Payable regression, PQU 10.0.48"). The name
is saved in the run's own file (`runs/<id>.json` for a plan run, `testruns/<file>.json` for a suite
or test case run), so the CLI keeps it too, and `trxa init-plan-run`/`init-run` take `--name` for
runs started outside Studio. A run without a name is listed by what it ran. **Rename** on the
run's page changes or clears it (not while a suite or test case run is still in progress).

**A run's page** opens from the table (or a link). The breadcrumb goes back to the table, and the
arrows step to the previous/next run in the table's current order. The header shows the name,
status, what was run and when, the environment, company, depth, who ran it and how long it took,
then a progress bar ("6 of 8 test cases finished"). **Stop run** appears while an agent run is in
progress. Below that:

- **Results**: for a plan run, a summary of its suites by status and a table of suites; a suite row
  expands to that suite's test cases. For a suite or test case run, its test cases directly.
  Each test case shows its status badge (`Passed`/`Failed`/`Blocked`/`Skipped`/`Running`/
  `Pending`), captured `outputs[]`, and every `expectations[]` assertion with a Pass/Fail badge
  and the `actual` value observed -- the same evidence `replay-trxa-testplan` records with
  `record-result`. A "failures only" toggle narrows the list. Tokens/Cost tiles appear when test
  cases report `tokensUsed`/`costUsd` (see MANUAL.md §7.8).
- **Agent activity** (agent runs only): the failure or interruption reason if there was one, the
  activity log of everything the agent did, how many tool calls the run's permissions denied, and
  **Full log** to download the whole transcript.

**Copy summary** copies results as Markdown and rich HTML (status colours and Infolog severity
kept), titled with the run's name. **Export** downloads the same as `.md`, or **PDF** through the
browser's print dialog (always in the light theme). **Delete** on a suite removes that Test Run
file after a confirmation; a plan run's **Delete run** removes only the `runs/<id>.json` record,
not its suites' Test Run files.

**While a run is in progress**, wherever you are in Studio:

- The header's run activity button turns into a spinner. Its panel lists each run in progress with
  a progress bar, then the five most recent runs.
- When an agent run ends, a notification says how it went ("Accounts payable regression: failed,
  7 passed, 1 failed") with **View run**; it stays until you dismiss it.
- The table and an open run page refresh on their own every few seconds, so a run started from
  the CLI or another window appears without a manual refresh.

Results are a durable record of what a replay did, so nothing on this tab edits them; only the name
can change. To produce a new run, use **Run** on a plan, suite or test case (§8a), or the CLI
procedure in MANUAL.md §8 "Running a plan".

## 8a. Running with an AI agent

Test Cases, Suites and Plans each have a **Run** button that hands the item to a GitHub Copilot
agent, which replays it against the D365 session your `d365-erp-mcp` server is connected to.

- **Readiness**: Run is disabled, with an "Agent not ready" link, until Settings > Agent runs shows
  Copilot ready (the `trxa` command, Copilot CLI and sign-in, D365 MCP server, replay skill). The
  agent drives each run with `trxa`, so a missing `trxa` blocks runs; the Windows app includes its
  own. When the replay skill is missing, **Install skill** on its row installs it into the working
  folder (the same as `trxa init-skill`).
- **Dialog**: name the run (see §8, "Run names"), enter the environment and company you *expect*
  (the agent checks the live session first and stops if it differs), and choose the depth. **Navigation only** (default) blocks
  every D365 tool that can change data. **Full run** can save and post real records; choosing it
  and clicking Start is the confirmation.
- **Progress**: starting a run opens its page in Test Runs (§8), which shows its progress and the
  agent's activity live. **Stop run** cancels it. The run keeps going if you go elsewhere; the
  header's run activity button and a notification at the end keep you informed. Reloading the
  page re-attaches to a run still in progress.
- Only one run at a time. Writes outside the working folder and non-`trxa` shell commands are
  rejected. Run transcripts are saved under `agent-runs/` in the working folder (local only).
- **Agent choice**: GitHub Copilot (default) or Claude Code, whichever is ready. Claude Code runs
  in print mode with allow/deny tool lists enforcing the depth; it can't ask questions mid-run, so
  on an environment mismatch it stops and reports.
- Auth or connection failures show a plain message pointing at `copilot login` or Settings.
- A run cut off by closing or restarting Studio is shown as failed ("interrupted") rather than
  stuck on running. Stored transcripts cap each message at 4,000 characters and 2,000 events.
  Nothing from a run is sent anywhere except to the agent you chose.

## 9. Cross-cutting UI notes

- **Toasts**: every Save on an editable tab (Recordings/Test Cases/Fixtures/Suites/Plans) confirms
  with a small message that fades in at the top of the screen, holds briefly, then fades out on
  its own -- no need to dismiss it. Test Runs has nothing to save, so it never shows one.
- **Error banners**: validation failures (missing required fields, a malformed filter, a suite DAG
  cycle, etc.) show as a banner at the top of the detail pane, not a toast -- toasts are for
  success, banners are for something you need to fix.
- **List/detail split pane**: every editing tab shares the same layout -- a searchable/scrollable
  list on the left, the selected item's editor on the right. Test Runs is different: a full-width
  table, and each run opens on its own page. Nothing
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
