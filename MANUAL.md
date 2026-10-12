# TRXA-TOOL setup & usage manual

This is the step-by-step guide for setting up `trxa-tool` and using it to turn a library of
D365 Finance & Operations Task Recorder (`.axtr`) files into automated, sequenced, data-driven
tests that LLMs can execute against a real D365FO environment via `d365-erp-mcp`.

This document is the "how do I actually run this?" guide. For the browser UI, see
[STUDIO.md](STUDIO.md).

---

## 1. Mental model

Six artifacts make up the whole pipeline, each one building on the last. A raw screen recording
becomes a generic, mechanical test script; you turn that into a named test case with real inputs
and pass/fail criteria; concrete test data and a run sequence tie multiple test cases together
into a suite; related suites can be grouped into a plan for combined reporting; and running a
suite produces a durable, trackable result instead of something that only ever existed in a chat
transcript. Here's how those pieces connect:

```
📼  .axtr file
    This is the Task Recorder output, a recording file you get from D365FO's Task Recorder, or from an
    LCS Business Process Modeler library export.
     │
     │  trxa parse
     ▼
📋  Test Script   (testscripts/*.json)
    This file contains the mechanics: ordered steps, each with a suggested d365-erp-mcp tool call. Fully
    generic, never hardcodes a control name, form name, or business value.
     │
     │  trxa suggest-inputs   (optional, drafts a starting point)
     ▼
🧩  Test Case   (testcases/*.json)
    This is where you come in: you edit this file and define which values are real test inputs, and what a passing run
    looks like. This is the atomic pass/fail unit.
     │
     │  you also edit one or more...
     ▼
🗃️  Fixture   (testdata/*.json)
    These represent the concrete values for a test case's inputs (e.g. "basic" vs. "eu-company"), so the
    same test case can run against different data.
     │
     │  a suite ties a test case + a fixture together, and defines their sequence.
     ▼
🔗  Test Suite   (suites/*.json)
    In here you define which test cases run, in what order, and where each one's
    inputs come from (a fixture, a literal, or an earlier test case's output).
     │
     │  optionally, group related suites for combined reporting:
     ▼
📚  Test Plan   (plans/*.json)
    You use this to define a list of suites (e.g. every suite in "Accounts
    Receivable"), run and reported on together. No dependencies between suites;
    if two suites need to share data, they belong in one suite instead.
     │
     │  trxa init-run
     ▼
✅  Test Run   (testruns/*.json)
    Tracks pass/fail per test case, captured outputs, and
    assertion results. Named `<suiteId>-<runId>.json` so every run is kept, not
    overwritten, and a plan's suites can share one `runId` for combined reporting.
```

Claude or another LLM client (using the `replay-trxa-testplan` skill) is what actually drives
the last mile: it asks `trxa resolve-next` what to run, replays that test case's Test
Script using `d365-erp-mcp` with the resolved concrete inputs, checks its `expectations`, and
calls `trxa record-result` to persist the outcome.

Everything that is deterministic in nature (parsing, DAG ordering, cycle detection, binding
resolution, plan-level rollups) is driven by code.

Everything that requires an actual D365FO session (clicking, typing, querying, judging
pass/fail) is the LLM's job.

## 2. Prerequisites

Before you can run a full suite against a live environment, make sure you have each of the
following in place:

- **Node.js** ≥ 18.19 (`node --version`)
- **[Claude Cowork](https://claude.com/claude-code)** (or another agent runtime capable of
  driving MCP tools). This is what actually executes a test suite
- **The `d365-erp-mcp` MCP server**, connected to the D365 Finance & Operations environment you
  want to test against, available in the same Claude Cowork session/project.
- One or more `.axtr` files. Get these either by recording them yourself in D365FO (System
  administration → the Task Recorder add-in), or from an existing LCS Business Process Modeler
  library.

## 3. Installation

Install the package globally. This puts the `trxa` command on your `PATH`, and every command in
the rest of this manual is written that way:

```sh
npm install -g trxa-tool
trxa help       # confirms the install and lists every command
```

Then, in your own project folder (where your recordings, test cases and suites will live), run
once:

```sh
trxa init-skill
```

Claude Code and Codex discover project skills from their own folders (`.claude/skills/` and
`.agents/skills/`), not from inside an installed package. `init-skill` copies the
`replay-trxa-testplan` skill (step 7.7 below) into both, so whichever agent you use finds it.

Studio, the browser UI (see [STUDIO.md](STUDIO.md)), is included in the package. There's nothing
extra to install; start it with `trxa studio`.

## 4. Directory layout

Directory-by-directory contents are below; which of these get committed to git (and why some are
deliberately ignored by default) is covered in [Data hygiene](#5-data-hygiene).

| Directory | Contents |
|---|---|
| `recordings/` | Raw `.axtr` recordings |
| `testscripts/` | Generated Test Script JSON (output of `parse`) |
| `testcases/` | Test Case JSON, the reusable, shareable test library |
| `testdata/` | Fixture JSON, concrete input values |
| `suites/` | Test Suite JSON, sequencing/DAGs |
| `plans/` | Test Plan JSON, groups of suites for combined reporting |
| `testruns/` | Generated Test Run JSON per run |

## 5. Data hygiene

Read this before uploading a real `.axtr` library. A `.axtr` recording captures whatever the
user interacted with when the recording was made (real account numbers, journal entries,
amounts, customer names, etc.). `testscripts/*.json` carries the same data forward, and
`suggest-inputs`'s draft `example` values are copied verbatim from the recording too.

If you keep your project in git, consider:

- **Ignoring `recordings/` and `testdata/` by default** in `.gitignore`, and allow-listing only
  files you've reviewed and know to be safe to share (for example, recordings made against
  standard demo data).
- **Ignoring `testscripts/` and `testruns/` entirely.** Test scripts are fully derived from a
  recording (regenerate them with `parse`), and test runs are generated output.
- **Reviewing the `example` fields** of any `testcases/*.json` you author (by hand or via
  `suggest-inputs`) before committing. Replace real business data with a placeholder of your
  choice.
- **Keeping `analysis/` local.** `analyze-library`'s `library-report.json` replaces every
  recorded value with an opaque token, so it's safe to share with a cloud LLM, but
  `scrubbed-values.csv` maps those tokens back to the real values. Don't open it anywhere an LLM
  can read it.

## 6. Full CLI reference

Run everything as `trxa <command> ...`. Running a command with no further arguments prints its
specific usage.

| Command | Purpose |
|---|---|
| `parse <file.axtr> [-o out.testscript.json]` | Extract + normalize a recording into a Test Script |
| `suggest-inputs <file.axtr> [-o draft.json]` | Draft candidate `inputs` for a Test Case from a recording |
| `validate-testcase <testcase.json>` | Validate a Test Case file |
| `validate-fixture <fixture.json>` | Validate a Fixture file |
| `validate-suite <suite.json>` | Validate a Test Suite, including DAG cycle/unknown-id checks |
| `plan-suite <suite.json>` | Print a suite's topological execution order |
| `init-skill [--force]` | Copy the `replay-trxa-testplan` skill into this project's `.claude/skills/` and `.agents/skills/`, so Claude Code and Codex both pick it up automatically -- run this once per project; refuses to overwrite an existing copy at either location unless `--force` is passed |
| `init-run <suite.json> -o testruns/<suiteId>-<runId>.json` | Create a fresh Test Run, one `"pending"` entry per suite entry |
| `resolve-next <suite.json> <testrun.json> --testcases-dir <dir> [--fixtures-dir <dir>]` | Get the next runnable test case (inputs already resolved), or `blocked`/`waiting`/`complete` |
| `measure-tokens --since <isoTimestamp> --until <isoTimestamp> [--transcript <path>]` | Sum real token usage from Claude Code's own session log for a time window (mostly for manual/debugging use -- `record-result` already calls this internally; only reliable when no other Claude Code session is active against this project, see "Token/cost reporting" below) |
| `record-result <testrun.json> <testCaseId> --status passed\|failed\|blocked --outputs '{...}' [--assertions '[...]']` | Persist a test case's outcome. Every JSON flag has a `-file` form (`--outputs-file <path>`, `--assertions-file`, `--inputs-file`, `--messages-file`); see [Windows PowerShell](#windows-powershell-pass-json-through-files) below |
| `validate-plan <plan.json>` | Validate a Test Plan, including that every referenced suite exists and is itself valid |
| `init-plan-run <plan.json> -o runs/` | Create a fresh PlanRun record for a plan, auto-generating its id -- do this once per plan execution, before running any of its suites |
| `plan-status <plan.json> <run.json> --testruns-dir <dir>` | Aggregate pass/fail across every suite in a plan for one PlanRun |
| `analyze-library <recordingsDir> [--report ...] [--csv ...]` | Privacy-safe coverage and gap analysis across a whole `recordings/` library (see [Data hygiene](#5-data-hygiene) for its two output files) |
| `studio` | Start the Studio browser UI in the background and open `http://localhost:5173` (see [STUDIO.md](STUDIO.md)) |
| `studio stop` | Stop the backgrounded Studio server + client started by `studio` |
| `export -o <backup.zip>` | Bundle `recordings/`/`testcases/`/`testdata/`/`testscripts/`/`suites/`/`plans/`/`testruns/` into one zip |
| `import <backup.zip> [--force]` | Unpack a library backup zip; refuses on filename collisions unless `--force` |

Every command exits non-zero and prints a specific, descriptive error on invalid input, nothing
fails silently.

### Windows PowerShell: pass JSON through files

Windows PowerShell 5.1 can silently corrupt JSON passed inline as a command-line argument before
`trxa` ever receives it: it strips the double quotes from JSON without spaces, and splits JSON
with spaces into several arguments. Either way `record-result` fails with a JSON syntax error even
though the command looks correct.

Use the `-file` form of each JSON flag instead: write the JSON to a file and pass its path.

```powershell
[IO.File]::WriteAllText("$PWD\outputs.json", '{"PurchId":"PO-0001"}')
trxa record-result testruns/my-run.json create-order --status passed --outputs-file outputs.json
```

Write the file without a byte-order mark, as above. In PowerShell 5.1, `Set-Content -Encoding utf8`
and `Out-File` add one, and the JSON then fails to parse.

This works the same in every shell, so it's a safe default anywhere. The replay skill already uses
it.

---

## 7. Step-by-step: building your first test suite

This walks through a small two-test-case suite as a self-contained tutorial, using a generic
`create-order`/`post-packing-slip` example so the steps apply to any scenario.

### 7.1 Parse your recording

```sh
trxa parse recordings/my-recording.axtr -o testscripts/my-recording.testscript.json
```

Open the output and skim `steps[]`. Each step has a `hint`, the suggested `d365-erp-mcp` tool
call, with a `confidence` of `"high"`, `"medium"`, or `"low"`. Low-confidence steps are almost
always "find and select a record" steps, because the recording only captured a screen position,
not a durable business key; those are exactly the steps that need to become test **inputs**.

### 7.2 Draft candidate inputs

```sh
trxa suggest-inputs recordings/my-recording.axtr -o testcases/my-recording.draft.json
```

This scans the Test Script and proposes one candidate per recorded field value
(`PropertyUserAction`) and one per low-confidence record-selection step. Each candidate has a
`bindTo` telling you exactly which step/control it plugs into, plus (for field values) the
recorded `example`; review and scrub that before it goes anywhere shared. This draft is *not*
a valid Test Case yet; it's the `inputs` array to copy into one.

### 7.3 Author the Test Case

Create `testcases/<your-id>.json`:

```jsonc
{
  "id": "create-order",                              // unique within a suite
  "name": "Select customer and open their sales order",
  "description": "...",
  "testScriptPath": "../testscripts/my-recording.testscript.json",  // relative to this file
  "inputs": [
    {
      "name": "customerAccount",                     // referenced by suite inputBindings
      "description": "...",
      "example": "US-001",                           // optional, from suggest-inputs
      "required": true,
      "bindTo": {
        "kind": "grid-row-selector",                 // or "kind": "control" + "controlName"
        "stepSequence": 21,
        "note": "In the list, find and select the desired record."
      }
    }
  ],
  "outputs": [
    {
      "name": "salesOrderId",
      "kind": "odata",                                // preferred when there's a solid business key to filter by
      "entitySet": "SalesOrderHeadersV2",
      "filter": "CustomerAccount eq '{{customerAccount}}'",  // {{name}} = resolved input value
      "field": "SalesOrderNumber"
    }
    // alternative, e.g. for a value only visible on screen (an auto-assigned ID Task Recorder
    // never captured because nobody clicked it) -- free at replay time, no extra query:
    // { "name": "...", "kind": "uiField", "afterStepSequence": N, "controlName": "..." }
    // if the control name isn't known yet, use "hint" instead (plain English); the replaying LLM
    // resolves it against live form state and writes the exact controlName back for future runs:
    // { "name": "...", "kind": "uiField", "afterStepSequence": N, "hint": "..." }
  ],
  "expectations": [
    {
      "description": "No error dialog appeared",
      "checkAfterStepSequence": 46,
      "kind": "uiMessage",                            // fallback when there's no entity to query
      "severity": "Error",
      "pattern": "",                                   // plain substring match; empty = any message of this severity
      "shouldExist": false
    }
    // preferred alternative:
    // { "description": "...", "checkAfterStepSequence": N, "kind": "odata", "entitySet": "...",
    //   "filter": "...", "expectedField": "...", "expectedValue": "...", "comparator": "equals" }
  ]
}
```

Field reference:

| Field | Type | Notes |
|---|---|---|
| `id`, `name`, `description` | `string` | `id` must be unique within any suite that references it |
| `testScriptPath` | `string` | Path to the Test Script JSON, relative to this file |
| `inputs[].bindTo.kind` | `"control"` \| `"grid-row-selector"` | `"control"` needs `controlName`; `"grid-row-selector"` needs a `note` (why/what to search for) |
| `outputs[].kind` | `"odata"` \| `"uiField"` | `"odata"` needs `entitySet`/`filter`/`field`; `"uiField"` needs `afterStepSequence` plus at least one of `controlName` (exact, free at replay time) or `hint` (plain English, for when the control name isn't known -- the replaying LLM resolves it against live form state and is expected to write the resolved `controlName` back) |
| `expectations[].kind` | `"odata"` \| `"uiMessage"` | `"odata"` needs `entitySet`/`filter`/`expectedField`/`expectedValue`/`comparator` (`equals`\|`notEquals`\|`contains`\|`greaterThan`\|`lessThan`); `"uiMessage"` needs `severity` (`Info`\|`Warning`\|`Error`)/`pattern` (plain substring of the message text, not a regex -- empty matches any message of that severity)/`shouldExist` (should a matching message be present, not whether a record exists) |

`filter` and `expectedValue` may contain `{{inputName}}` placeholders; Claude substitutes the
resolved input value before querying, as plain string substitution (not a templating engine).

Validate as you go:

```sh
trxa validate-testcase testcases/create-order.json
```

### 7.4 Author a Fixture (concrete test data)

Create `testdata/<your-id>.<fixture-name>.json`:

```json
{
  "testCaseId": "create-order",
  "name": "basic",
  "values": { "customerAccount": "US-001" }
}
```

You can have multiple fixtures per test case (e.g. `basic`, `eu-company`); the suite picks
which one to use per run. The fixture is not part of the test case itself; it's chosen at the
suite level (see [Author the Test Suite](#75-author-the-test-suite)), which is what lets the
same Test Case run against different data.

```sh
trxa validate-fixture testdata/create-order.basic.json
```

### 7.5 Author the Test Suite

```json
{
  "id": "packing-slip-flow",
  "name": "Create order then post its packing slip",
  "entries": [
    {
      "testCaseId": "create-order",
      "dependsOn": [],
      "inputBindings": {
        "customerAccount": { "kind": "fixture", "fixtureName": "basic", "param": "customerAccount" }
      }
    },
    {
      "testCaseId": "post-packing-slip",
      "dependsOn": ["create-order"],
      "inputBindings": {
        "salesOrderId": { "kind": "testCaseOutput", "testCaseId": "create-order", "outputName": "salesOrderId" }
      }
    }
  ]
}
```

`inputBindings` per param, one of:

| `kind` | Fields | Meaning |
|---|---|---|
| `"fixture"` | `fixtureName`, `param` | Pull a value from a named Fixture's `values` |
| `"testCaseOutput"` | `testCaseId`, `outputName` | Pull a value from an **already-passed** upstream test case's captured output; this is what threads data through a sequence |
| `"literal"` | `value` | A hardcoded value right in the suite file (fine for constants, not for real business data) |
| `"autogenerated"` | *(none)* | No value here at all -- `resolve-next` reports this param in `autogeneratedParams` (with the input's own `description`/`example` as guidance) instead of resolving it, and the orchestrating LLM replaying the suite invents a value at that point, typically for a business key that just needs to be unique per run |

An entry can also declare `runIf` to skip it conditionally, e.g. added to `post-packing-slip`
above:

```json
"runIf": { "testCaseId": "create-order", "outputName": "currency", "comparator": "equals", "value": "USD" }
```

`testCaseId` must be one of this entry's own `dependsOn` -- by the time `runIf` is evaluated that
dependency has already passed, so the condition compares its captured *output*, not its status
(which would always be `"passed"` here, so checking it would be pointless). A false condition
records the entry as `"skipped"` instead of running it -- not a pass, not a failure, and (like
`"blocked"`) a downstream entry that depends on a skipped one is itself reported `"blocked"`, not
left waiting forever.

```sh
trxa validate-suite suites/packing-slip-flow.json
trxa plan-suite suites/packing-slip-flow.json
```

`validate-suite` also runs the topological sort, so it'll immediately tell you about a cycle or
a `dependsOn` referencing a `testCaseId` that doesn't exist in the suite. (`plan-suite` here
means "print this suite's execution plan", a pre-existing command name that predates, and is
unrelated to, the Test Plan concept covered in
[Grouping suites into a Test Plan](#8-grouping-suites-into-a-test-plan).)

### 7.6 Start a run

```sh
trxa init-run suites/packing-slip-flow.json -o testruns/packing-slip-flow-$(date +%s).json
```

### 7.7 Run it: hand this to your LLM Client (Claude Cowork)

Open a Claude Cowork session with `d365-erp-mcp` connected, and ask it to run the suite (invoking
the `replay-trxa-testplan` skill; it will pick this up automatically from context, or you can
say so explicitly: *"use the replay-trxa-testplan skill to run suites/packing-slip-flow.json
against testruns/packing-slip-flow-....json"*).

What the LLM actually does, per test case, in a loop (see the skill's "Running as part of a Test
Suite" section for the authoritative version):

1. `resolve-next` → get the next test case with inputs already resolved (or `blocked`/`waiting`/`complete`)
2. Replay its `testScriptPath` per the normal per-step procedure, substituting each `bindTo`'d
   input with its resolved value instead of the recorded one
3. Evaluate `expectations` (OData query preferred, UI message fallback)
4. Capture `outputs` the same way
5. `record-result` with status + outputs + assertion results
6. Repeat until `resolve-next` reports `"complete"`

### 7.8 Read the result

```sh
cat testruns/packing-slip-flow-....json
```

`results[testCaseId].status` is `"passed"`, `"failed"`, `"blocked"` (an upstream dependency
failed or was skipped), or `"skipped"` (its own `runIf` evaluated false -- see §7.5). `outputs`
shows what was captured. `assertionResults` shows each expectation's actual observed value; this
is your expected-vs-actual audit trail, not something living only in a
chat transcript.

### Token/cost reporting

Each result can optionally carry `tokensUsed` (a raw token count) and/or `costUsd` (a dollar
figure). Neither is computed by this CLI itself -- it never talks to the model, so it has no
ground truth of its own -- but they aren't guesses either:

- `tokensUsed` is measured automatically -- `record-result` calls `measure-tokens` internally
  whenever it isn't passed `--tokens` explicitly and the test case has a known start time, no
  action needed from the orchestrating LLM or the person driving the CLI by hand. It reads the
  real `usage` data Claude Code writes to `~/.claude/projects/.../<session>.jsonl` on every turn
  and sums it for the window between this test case's `"running"` transition and the
  `record-result` call.

  **Treat this as a rough order-of-magnitude signal, not a trustworthy number.** Live-tested
  2026-08-20 by running two suites headlessly in parallel: the reported total was inflated 24% to
  98% (nearly double) versus each session's own authoritative usage total, because the underlying
  heuristic (it reads whichever session log was modified most recently) occasionally attributes another Claude Code session's turns to this one whenever more
  than one session is active against this project directory at the same time. That's not a rare
  edge case -- it's close to the default way this actually gets used (parallel suites, or a Desktop
  session left open while a background job runs). If you need a real number instead of a rough
  signal, a `claude -p --output-format json` invocation reports its own authoritative
  `total_cost_usd`/`usage` for exactly that invocation, with no cross-session risk at all -- that's
  a materially better source than this field, if you're already running headlessly.
  If no transcript can be found at all, `tokensUsed` is simply left unset for that result rather
  than guessed. This is a deliberate Claude-Code-only bonus, not a core requirement -- the whole
  `resolve-next`/`record-result` pipeline and the replay skill's instructions are meant to work
  with any sufficiently capable agentic LLM driving them, and no part of pass/fail or captured outputs
  depends on this field. Running under a different agent, or under Claude Cowork (which executes
  on Anthropic's own servers rather than the local machine), simply leaves it unset -- there's no
  local session transcript to read either way, which is expected, not a bug.
- `costUsd` is left unset in the interactive-skill execution mode described in this manual --
  there's no dollar figure worth trusting without a real, current, cache-aware pricing table, and
  this CLI has no business maintaining one. The field exists for a future execution mode (e.g.
  headless `claude -p --output-format json`, which reports `total_cost_usd` itself) to populate
  directly once that path exists.

Studio's Test Runs page shows both, per test case and aggregated across a whole run, whenever at
least one result reports them -- see [STUDIO.md](STUDIO.md).

---

## 8. Grouping suites into a Test Plan

Once you have more than one suite covering a related business area (e.g. "Accounts Receivable"
spanning customer onboarding, order-to-cash, and collections), a Test Plan groups them for
combined scheduling/reporting without adding a dependency graph between suites (if two suites
truly need to share data, put those test cases in one suite instead).

Create `plans/<your-id>.json`:

```json
{
  "id": "accounts-receivable",
  "name": "Accounts Receivable regression pack",
  "description": "...",
  "suites": [
    { "suiteId": "packing-slip-flow", "suitePath": "../suites/packing-slip-flow.json" }
  ]
}
```

`suitePath` is relative to the plan file, same convention as a Test Case's `testScriptPath`.
Array order is just the suggested run order; there's no `dependsOn` at the plan level.

```sh
trxa validate-plan plans/accounts-receivable.json
```

This resolves and validates every referenced suite too (catching a typo'd path or a broken
suite immediately), not just the plan's own shape.

**Running a plan** starts with one `init-plan-run` call to get a PlanRun record (its `id` is what
ties every suite's Test Run together), then the same per-suite loop from
[Run it: hand this to your LLM Client](#77-run-it-hand-this-to-your-llm-client-claude-cowork),
repeated once per suite in the plan:

```sh
trxa init-plan-run plans/accounts-receivable.json -o runs/
# prints the generated PlanRun, e.g. { "id": "2026-08-25T14-30-00.000Z", "planId": "accounts-receivable", ... }
runId=2026-08-25T14-30-00.000Z
trxa init-run suites/packing-slip-flow.json -o testruns/packing-slip-flow-$runId.json
# ... run that suite to completion via the skill, same as step 7.7 ...
# ... repeat init-run + the skill loop for each other suite in the plan, same $runId ...
trxa plan-status plans/accounts-receivable.json runs/$runId.json --testruns-dir testruns
```

`plan-status` reads `testruns/<suiteId>-<runId>.json` for every suite in the plan and reports a
rollup: `overall` is `"all-passed"`, `"has-failures"`, `"incomplete"` (a suite hasn't finished
running yet), or `"missing-runs"` (a suite hasn't been run at all for this PlanRun), plus a
per-suite breakdown. It's a pure, read-only query; nothing is persisted beyond the individual
suites' own Test Run files, so you can call it as often as you like while suites are still in
progress.

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `validate-suite` throws `has a dependency cycle involving: ...` | Two entries depend on each other (directly or transitively) | Fix `dependsOn` in the listed entries |
| `validate-suite` throws `depends on unknown testCaseId "..."` | Typo in `dependsOn`, or referenced entry missing from `entries[]` | Add the entry or fix the id |
| `validate-suite` throws `runIf references "..." which must be one of its own dependsOn` | A `runIf.testCaseId` isn't in that entry's own `dependsOn` | Add it to `dependsOn`, or point `runIf` at a test case that's already a direct dependency |
| `resolveInputs` throws `no inputBindings entry for required param "..."` | A Test Case input has no matching entry in the suite's `inputBindings` | Add a binding for that param name in the suite file |
| `resolveInputs` throws `has not passed yet` | A `testCaseOutput` binding points at a test case that hasn't run (or failed) | Check `dependsOn` includes that test case, and that it actually passed |
| `resolve-next` reports `"blocked"` | An upstream test case failed, or was itself skipped | Fix the upstream failure, or accept that dependent is blocked for this run |
| `resolve-next` reports `"skipped"` | An entry's `runIf` evaluated false against its dependency's captured output | Not an error -- this is the conditional-skip feature working as intended; nothing to fix |
| `resolve-next` reports `"waiting"` | A test case is stuck in `"running"` status without a matching `record-result` call | Finish that test case's `record-result` call |
| A step's hint has `confidence: "low"` | The recording captured a row position, not a business key | Expected. This is exactly what a Test Case input with `bindTo.kind: "grid-row-selector"` exists to solve |
| `validate-plan` throws `entry declares suiteId "..." but ... has id "..."` | The `suiteId` in a plan entry doesn't match the `id` inside the suite file it points at | Fix whichever one is wrong; they must match |
| `plan-status` reports a suite as `"missing"` | No `testruns/<suiteId>-<runId>.json` exists for that PlanRun's id | You haven't run that suite yet for this PlanRun; `init-run` + run it, using the same id as the rest of the plan |
| `plan-status` throws `Plan run at ... belongs to plan "..."` | The `run.json` passed in was created via `init-plan-run` against a *different* plan | Pass the PlanRun file that was actually created for this plan |

## 10. Best practices

- Never hand-guess business data during a replay; if a value should vary per run, it belongs in
  a Fixture, not typed in on the fly.
- Prefer `kind: "odata"` for `expectations`/`outputs` whenever the outcome maps to a queryable
  entity: it's far more robust than parsing UI text, and it's what actually proved the concept
  (an inventory-shortage defect was caught this way during the initial dry run).
- Run `validate-testcase`/`validate-fixture`/`validate-suite`/`validate-plan` before every test
  run; they're free and catch authoring mistakes deterministically instead of burning through
  tokens on a live D365FO session because of a typo.
- Review `example` values and any literal test data before committing; see
  [Data hygiene](#5-data-hygiene).
- One Test Case per distinct business scenario, reused across multiple suites/fixtures as
  needed, rather than one giant do-everything test case.
- Reach for a Test Plan once you have several suites covering one business area, rather than
  running each suite ad hoc; it's free composition/reporting with no added authoring cost per
  suite.
