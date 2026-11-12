# TRXA-TOOL manual

This manual explains how to use the `trxa` command line to turn Dynamics 365 Finance and Supply
Chain Management Task Recorder recordings (`.axtr` files) into automated, data-driven tests, and
how an AI agent runs those tests against an environment through the Dynamics 365 ERP MCP server.
It also documents every file format that TRXA-TOOL reads and writes.

If you prefer a visual workspace, [TRXA Studio](STUDIO.md) does everything in this manual without
editing files by hand. Studio and the command line use the same files, so you can switch between
them at any time.

## How TRXA-TOOL works

TRXA-TOOL works with seven kinds of files. Each one builds on the one before it.

```text
.axtr recording            A Task Recorder file, recorded in Dynamics 365 or exported
     |                     from a Business Process Modeler library in Lifecycle Services.
     |  trxa parse
     v
Test script                The recording's ordered steps, each with a suggested MCP tool call.
(testscripts/*.json)       Generic: it never hard-codes a business value.
     |
     |  trxa suggest-inputs (optional: drafts a starting point)
     v
Test case                  One pass/fail scenario. You define its inputs, the values it
(testcases/*.json)         captures (outputs), and what a passing run looks like (expectations).
     |
     v
Fixture                    A named set of test data, such as one company or one scenario,
(testdata/*.json)          with input values for every test case it covers.
     |
     |  a suite combines test cases with a fixture and defines their order
     v
Test suite                 Which test cases run, in what order, and where each input
(suites/*.json)            gets its value: a fixture, a literal, or an earlier test case's output.
     |
     |  optional: group related suites for combined reporting
     v
Test plan                  A list of suites that run and report together, such as every
(plans/*.json)             suite for Accounts receivable. Suites in a plan don't share data.
     |
     |  trxa init-run
     v
Test run                   Pass or fail per test case, captured outputs, and the result of
(testruns/*.json)          each check. Every run is kept, never overwritten.
```

TRXA-TOOL handles everything that has a fixed answer: reading recordings, ordering test cases,
detecting dependency loops, resolving input values and rolling up plan results. An AI agent, using
the `replay-trxa-testplan` skill, does everything that needs a live Dynamics 365 session. It asks
`trxa resolve-next` which test case to run, replays that test case's test script through the
`d365-erp-mcp` server, checks its expectations, and calls `trxa record-result` to save the outcome.

## Prerequisites

Before you run a suite against an environment, make sure that you have:

- **Node.js** 18.19 or later. To check, run `node --version`.
- **An AI agent that can use MCP tools**, such as GitHub Copilot CLI, Claude Code, Claude Cowork or
  Codex.
- **The `d365-erp-mcp` MCP server**, connected to the environment that you want to test and
  available to your agent. For setup instructions, including the `trxa mcp-bridge` connector, see
  [Set up TRXA Studio for agent runs](SETUP.md).
- **One or more `.axtr` recordings.** Record them in Dynamics 365 with Task recorder, or export
  them from a Business Process Modeler library in Lifecycle Services.

## Install TRXA-TOOL

1. Install the package globally. This puts the `trxa` command on your `PATH`. Every command in
   this manual uses it.

   ```sh
   npm install -g trxa-tool
   ```

1. Confirm the installation. The command lists every `trxa` command.

   ```sh
   trxa help
   ```

1. In your project folder, where your recordings, test cases and suites will be, install the
   replay skill:

   ```sh
   trxa init-skill
   ```

   Agents find project skills in their own folders, not inside an installed package. `init-skill`
   copies the `replay-trxa-testplan` skill into `.claude/skills` (used by Claude Code and GitHub
   Copilot CLI) and `.agents/skills` (used by Codex). Run it once per project.

TRXA Studio is included in the package. To start it, run `trxa studio`.

## Project folders

TRXA-TOOL expects these folders in your project. To decide which of them to commit to source
control, see [Data hygiene](#data-hygiene).

| Folder | Contents |
|---|---|
| `recordings` | Task Recorder recordings (`.axtr`) |
| `testscripts` | Test scripts, created by `trxa parse` |
| `testcases` | Test cases: your reusable test library |
| `testdata` | Fixtures: test data values |
| `suites` | Test suites: run order and data flow |
| `plans` | Test plans: groups of suites for combined reporting |
| `runs` | Plan run records, created by `trxa init-plan-run` |
| `testruns` | Test run results, one file per suite run |

## Data hygiene

Read this section before you add a library of real recordings. A recording captures whatever the
user worked with when it was made, such as real account numbers, journal entries, amounts and
customer names. Test scripts carry the same data forward, and the `example` values that
`suggest-inputs` drafts are copied directly from the recording.

If you keep your project in source control, consider these practices:

- **Ignore `recordings` and `testdata` by default** in `.gitignore`. Allow only files that you
  reviewed and know are safe to share, such as recordings made with standard demo data.
- **Ignore `testscripts` and `testruns` entirely.** You can regenerate test scripts from the
  recordings with `trxa parse`, and test runs are output.
- **Review the `example` fields** of every test case before you commit it, and replace real
  business data with placeholders.
- **Keep the `analysis` folder local.** The `library-report.json` file that `analyze-library`
  writes replaces every recorded value with a token, so it's safe to share with a cloud AI service.
  The `scrubbed-values.csv` file maps those tokens back to the real values, so never open it where
  an AI service can read it.

## Command reference

Run each command as `trxa <command>`. To see a command's options, run it with no arguments.

| Command | Description |
|---|---|
| `parse <file.axtr> [-o out.testscript.json]` | Reads a recording and writes its test script. |
| `suggest-inputs <file.axtr> [-o draft.json]` | Drafts candidate test case inputs from a recording. |
| `validate-testcase <testcase.json>` | Checks a test case file. |
| `validate-fixture <fixture.json>` | Checks a fixture file. |
| `migrate-fixtures <testdata-dir> [--dry-run]` | Merges fixture files in the old one-test-case format into one file per fixture. See [Create a fixture](#step-4-create-a-fixture). |
| `validate-suite <suite.json>` | Checks a suite file, including dependency loops and unknown test case IDs. |
| `plan-suite <suite.json>` | Prints the order that a suite's test cases run in. |
| `validate-plan <plan.json>` | Checks a plan file, and that every suite it refers to exists and is valid. |
| `init-skill [--force]` | Copies the `replay-trxa-testplan` skill into the project's `.claude/skills` and `.agents/skills` folders. Doesn't overwrite an existing copy unless you add `--force`. |
| `mcp-bridge` | Connects an agent to an environment's Dynamics 365 ERP MCP server. The agent starts it; you don't run it yourself. See [Step 5](SETUP.md#step-5-connect-the-agent-to-dynamics-365) of the setup article. |
| `init-plan-run <plan.json> -o runs/ [--name "<run name>"]` | Creates a plan run record with a generated ID. Run it once per plan run, before you run any of its suites. `--name` labels the run, for example `"AP regression, PQU 10.0.48"`. |
| `init-run <suite.json> -o testruns/<suiteId>-<runId>.json [--name "<run name>"]` | Creates a test run with every test case set to `pending`. `--name` labels the run. |
| `resolve-next <suite.json> <testrun.json> --testcases-dir <dir> [--fixtures-dir <dir>]` | Returns the next test case to run, with its inputs resolved, or reports `blocked`, `waiting` or `complete`. |
| `record-result <testrun.json> <testCaseId> --status passed\|failed\|blocked --outputs '{...}' [--assertions '[...]']` | Saves a test case's outcome. Each JSON option has a file form: `--outputs-file`, `--assertions-file`, `--inputs-file` and `--messages-file`. See [Pass JSON through files in Windows PowerShell](#pass-json-through-files-in-windows-powershell). |
| `measure-tokens --since <isoTimestamp> --until <isoTimestamp> [--transcript <path>]` | Adds up the tokens that Claude Code used in a time window. `record-result` already does this. See [Token and cost reporting](#token-and-cost-reporting). |
| `plan-status <plan.json> <run.json> --testruns-dir <dir>` | Rolls up pass and fail across every suite in a plan for one plan run. |
| `analyze-library <recordingsDir> [--report ...] [--csv ...]` | Reports coverage and gaps across a whole recordings library without exposing recorded values. See [Data hygiene](#data-hygiene). |
| `studio` | Starts TRXA Studio in the background and opens `http://127.0.0.1:4321` in your browser. See [Use TRXA Studio](STUDIO.md). |
| `studio stop` | Stops TRXA Studio. |
| `export -o <backup.zip>` | Bundles the `recordings`, `testcases`, `testdata`, `testscripts`, `suites`, `plans` and `testruns` folders into one zip file. |
| `import <backup.zip> [--force]` | Restores a library backup. Stops if a file already exists, unless you add `--force`. |

If an input is invalid, every command prints a specific error and exits with a nonzero code.

### Pass JSON through files in Windows PowerShell

Windows PowerShell 5.1 can corrupt JSON that you pass directly as a command-line argument, before
`trxa` receives it. It removes the double quotes from JSON without spaces, and splits JSON with
spaces into several arguments. Either way, `record-result` fails with a JSON syntax error even
though the command looks correct.

Instead, use the file form of each JSON option. Write the JSON to a file, and then pass the file's
path:

```powershell
[IO.File]::WriteAllText("$PWD\outputs.json", '{"PurchId":"PO-0001"}')
trxa record-result testruns/my-run.json create-order --status passed --outputs-file outputs.json
```

Write the file without a byte order mark, as in this example. In PowerShell 5.1,
`Set-Content -Encoding utf8` and `Out-File` add one, and the JSON then fails to parse.

The file form works the same in every shell, so it's a safe default anywhere. The replay skill
already uses it.

## Tutorial: Build and run your first suite

This tutorial builds a suite of two test cases: one creates a sales order, and the next posts its
packing slip. The names are generic, so the steps apply to any scenario.

### Step 1: Parse your recording

```sh
trxa parse recordings/my-recording.axtr -o testscripts/my-recording.testscript.json
```

Open the output and review `steps`. Each step has a `hint`, the suggested `d365-erp-mcp` tool call,
with a `confidence` of `high`, `medium` or `low`. Low-confidence steps are almost always steps that
find and select a record. The recording captured only a position on the screen, not a business
key, so those steps need to become test case inputs.

### Step 2: Draft candidate inputs

```sh
trxa suggest-inputs recordings/my-recording.axtr -o testcases/my-recording.draft.json
```

The command proposes one candidate input for each value typed in the recording
(`PropertyUserAction`) and one for each low-confidence record selection. Each candidate has a
`bindTo` that names the step and control it applies to and, for typed values, the recorded
`example`. Review the examples and remove any real business data before you share them. The draft
isn't a valid test case yet. It's the `inputs` array that you copy into one.

### Step 3: Create the test case

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
      "kind": "odata",                                // preferred when a reliable business key exists
      "entitySet": "SalesOrderHeadersV2",
      "filter": "CustomerAccount eq '{{customerAccount}}'",  // {{name}} = resolved input value
      "field": "SalesOrderNumber"
    }
    // Alternative for a value that's only on screen, such as a number assigned by the system.
    // No extra query is needed at replay time:
    // { "name": "...", "kind": "uiField", "afterStepSequence": N, "controlName": "..." }
    // If you don't know the control name yet, use "hint" (plain English). The agent finds the
    // control during the run and writes its exact controlName back for later runs:
    // { "name": "...", "kind": "uiField", "afterStepSequence": N, "hint": "..." }
  ],
  "expectations": [
    {
      "description": "No error dialog appeared",
      "checkAfterStepSequence": 46,
      "kind": "uiMessage",                            // use when there's no entity to query
      "severity": "Error",
      "pattern": "",                                   // plain text match; empty = any message of this severity
      "shouldExist": false
    }
    // Preferred alternative:
    // { "description": "...", "checkAfterStepSequence": N, "kind": "odata", "entitySet": "...",
    //   "filter": "...", "expectedField": "...", "expectedValue": "...", "comparator": "equals" }
  ]
}
```

The following table describes the fields.

| Field | Type | Description |
|---|---|---|
| `id`, `name`, `description` | `string` | `id` must be unique within every suite that uses the test case. |
| `testScriptPath` | `string` | The path to the test script, relative to the test case file. |
| `inputs[].bindTo.kind` | `"control"` or `"grid-row-selector"` | `"control"` needs `controlName`. `"grid-row-selector"` needs a `note` that says what to search for. |
| `outputs[].kind` | `"odata"` or `"uiField"` | `"odata"` needs `entitySet`, `filter` and `field`. `"uiField"` needs `afterStepSequence` and at least one of `controlName` (exact, no query needed) or `hint` (plain English, for when you don't know the control name; the agent writes the control name back after it finds it). |
| `expectations[].kind` | `"odata"` or `"uiMessage"` | `"odata"` needs `entitySet`, `filter`, `expectedField`, `expectedValue` and `comparator` (`equals`, `notEquals`, `contains`, `greaterThan` or `lessThan`). `"uiMessage"` needs `severity` (`Info`, `Warning` or `Error`), `pattern` (plain text that the message contains, not a regular expression; empty matches any message of that severity) and `shouldExist` (whether a matching message must appear). |

`filter` and `expectedValue` can contain `{{inputName}}` placeholders. The agent replaces each one
with the resolved input value before it runs the query. The replacement is plain text, not a
template language.

Check the file as you work:

```sh
trxa validate-testcase testcases/create-order.json
```

### Step 4: Create a fixture

A fixture is one set of test data, such as one company or one scenario, with values for every test
case it covers. Create `testdata/<fixture-name>.json`:

```json
{
  "name": "basic",
  "description": "US company, standard customer",
  "testCases": {
    "create-order": { "customerAccount": "US-001" },
    "post-packing-slip": { "warehouse": "11" }
  }
}
```

Suites refer to the fixture by `name`, which must match the file name. A name can contain letters,
digits, spaces, periods, underscores and hyphens, and fixture names must be unique, ignoring case.
A test case can have values in several fixtures, such as `basic` and `eu-company`. The suite
chooses which fixture to use (see [Step 5](#step-5-create-the-suite)), so the same test case can
run with different data.

```sh
trxa validate-fixture testdata/basic.json
```

> [!NOTE]
> Earlier versions stored one test case's values per fixture file
> (`testdata/<test-case-id>.<fixture-name>.json`, with `testCaseId` and `values`). Those files
> still work, because `resolve-next` merges files with the same `name`. To convert them to the
> current format, run `trxa migrate-fixtures testdata --dry-run` to preview the changes, and then
> run it again without `--dry-run`. The command checks everything before it writes. If two files
> hold values for the same test case under the same fixture name, it changes nothing and names
> both files.

### Step 5: Create the suite

Create `suites/<your-id>.json`:

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

Each input binding has one of these kinds:

| `kind` | Fields | The value comes from |
|---|---|---|
| `"fixture"` | `fixtureName`, `param` | The named fixture's values for this test case. |
| `"testCaseOutput"` | `testCaseId`, `outputName` | An output captured by an upstream test case that already passed. This is how data flows through a suite. |
| `"literal"` | `value` | A value written in the suite file. Use it for constants, not for real business data. |
| `"autogenerated"` | None | The agent. `resolve-next` lists the input in `autogeneratedParams`, with its `description` and `example` as guidance, and the agent invents a value during the run. Use it for values that only need to be unique, such as a new business key. |

An entry can also have a `runIf` condition that skips it. For example, add this to
`post-packing-slip`:

```json
"runIf": { "testCaseId": "create-order", "outputName": "currency", "comparator": "equals", "value": "USD" }
```

`testCaseId` must be one of the entry's own `dependsOn` test cases. When `runIf` is evaluated, that
dependency has already passed, so the condition compares one of its outputs, not its status. If
the condition is false, the entry is recorded as `skipped` instead of running. A skipped entry is
neither a pass nor a failure. Entries that depend on a skipped entry are reported as `blocked`.

Check the suite, and print its run order:

```sh
trxa validate-suite suites/packing-slip-flow.json
trxa plan-suite suites/packing-slip-flow.json
```

`validate-suite` immediately reports a dependency loop, or a `dependsOn` that names a test case
that isn't in the suite.

> [!NOTE]
> `plan-suite` prints a suite's execution plan. The name predates test plans and isn't related to
> them. For test plans, see [Group suites into a test plan](#group-suites-into-a-test-plan).

### Step 6: Start a run

```sh
trxa init-run suites/packing-slip-flow.json -o testruns/packing-slip-flow-$(date +%s).json
```

### Step 7: Ask your agent to run the suite

Open a session in your agent, with `d365-erp-mcp` connected, and ask it to run the suite. The agent
uses the `replay-trxa-testplan` skill. To be explicit, name the skill, for example: *"Use the
replay-trxa-testplan skill to run suites/packing-slip-flow.json against
testruns/packing-slip-flow-....json."*

For each test case, the agent repeats these steps. The skill's "Running as part of a Test Suite"
section is the authoritative version.

1. Runs `resolve-next` to get the next test case, with its inputs resolved, or a `blocked`,
   `waiting` or `complete` status.
1. Replays the test case's test script, using the resolved value for each input instead of the
   recorded one.
1. Checks the `expectations`. It prefers OData queries and falls back to messages on the form.
1. Captures the `outputs`.
1. Runs `record-result` with the status, the outputs and the result of each check.
1. Stops when `resolve-next` reports `complete`.

### Step 8: Read the results

Open the test run file, for example:

```sh
cat testruns/packing-slip-flow-....json
```

For each test case, `results[testCaseId].status` is one of these values:

| Status | Meaning |
|---|---|
| `passed` | Every expectation held. |
| `failed` | At least one expectation didn't hold, or a step couldn't be performed. |
| `blocked` | An upstream test case failed or was skipped, so this one didn't run. |
| `skipped` | The test case's own `runIf` condition was false. See [Step 5](#step-5-create-the-suite). |

`outputs` shows the captured values. `assertionResults` shows the value that the agent observed for
each expectation, so you have a record of expected and actual values that doesn't depend on a chat
transcript.

To review results visually, open **Test Runs** in [TRXA Studio](STUDIO.md#review-test-runs).

### Token and cost reporting

Each result can include `tokensUsed`, a token count, and `costUsd`, a cost in US dollars. The CLI
never talks to an AI model, so it doesn't calculate either value itself.

- **`tokensUsed`** is measured automatically. Unless you pass `--tokens`, `record-result` runs
  `measure-tokens` for the time between the test case's start and the `record-result` call. It
  reads the usage data that Claude Code writes to its session log in
  `~/.claude/projects/.../<session>.jsonl`.

  Treat this value as a rough estimate. In a test with two suites running in parallel, the reported
  totals were 24% to 98% higher than each session's own usage totals. The measurement reads the
  most recently changed session log, so when more than one Claude Code session works in the same
  project folder at the same time, it can count another session's usage. That situation is common,
  for example with suites running in parallel, or a desktop session left open during a background
  run. For an exact number, run Claude Code with `claude -p --output-format json`, which reports
  `total_cost_usd` and `usage` for that run only.

  If no session log is found, `tokensUsed` is left empty. The value is available only with Claude
  Code. Other agents, and Claude Cowork, which runs in the cloud, leave it empty. Pass or fail and
  captured outputs never depend on it.

- **`costUsd`** is left empty when an agent runs tests interactively, because a reliable cost needs
  a current pricing table that this CLI doesn't maintain. The field is for a future mode, such as
  `claude -p --output-format json`, that reports its own cost.

When at least one result has these values, TRXA Studio shows them for each test case and as totals
for the run.

## Group suites into a test plan

When you have several suites for one business area, such as customer onboarding, order to cash and
collections for Accounts receivable, a test plan groups them so that you can run and report on them
together. A plan doesn't add dependencies between suites. If two suites need to share data, put
their test cases in one suite.

1. Create `plans/<your-id>.json`:

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

   `suitePath` is relative to the plan file, the same as a test case's `testScriptPath`. The order
   of `suites` is the suggested run order. Plans have no `dependsOn`.

1. Check the plan. The command also checks every suite that the plan refers to, so it catches a
   wrong path or a broken suite right away.

   ```sh
   trxa validate-plan plans/accounts-receivable.json
   ```

### Run a plan

A plan run starts with one `init-plan-run` call. Its result has an `id` that ties every suite's
test run together. Then run each suite as in
[Step 7](#step-7-ask-your-agent-to-run-the-suite), using that same ID:

```sh
trxa init-plan-run plans/accounts-receivable.json -o runs/
# prints the new plan run, for example { "id": "2026-08-25T14-30-00.000Z", "planId": "accounts-receivable", ... }
runId=2026-08-25T14-30-00.000Z
trxa init-run suites/packing-slip-flow.json -o testruns/packing-slip-flow-$runId.json
# ... run that suite to completion with the skill, as in step 7 ...
# ... repeat init-run and the skill for each other suite in the plan, with the same $runId ...
trxa plan-status plans/accounts-receivable.json runs/$runId.json --testruns-dir testruns
```

`plan-status` reads `testruns/<suiteId>-<runId>.json` for every suite in the plan and reports a
per-suite breakdown and an `overall` status:

| `overall` | Meaning |
|---|---|
| `all-passed` | Every suite passed. |
| `has-failures` | At least one suite has a failed or blocked test case. |
| `incomplete` | At least one suite is still running. |
| `missing-runs` | At least one suite hasn't run yet in this plan run. |

`plan-status` only reads files, so you can run it as often as you like while suites are still
running.

## Troubleshoot

| Symptom | Cause | Solution |
|---|---|---|
| `validate-suite` reports `has a dependency cycle involving: ...` | Two entries depend on each other, directly or through other entries. | Fix `dependsOn` in the listed entries. |
| `validate-suite` reports `depends on unknown testCaseId "..."` | A `dependsOn` value is misspelled, or the entry it names isn't in `entries`. | Add the entry, or fix the ID. |
| `validate-suite` reports `runIf references "..." which must be one of its own dependsOn` | A `runIf.testCaseId` isn't in that entry's `dependsOn`. | Add it to `dependsOn`, or point `runIf` at a direct dependency. |
| `resolve-next` reports `no inputBindings entry for required param "..."` | A required test case input has no binding in the suite. | Add a binding for that input in the suite file. |
| `resolve-next` reports `has not passed yet` | A `testCaseOutput` binding points at a test case that hasn't run, or that failed. | Make sure `dependsOn` includes that test case, and that it passed. |
| `resolve-next` reports `blocked` | An upstream test case failed or was skipped. | Fix the upstream failure, or accept that the dependent test case is blocked for this run. |
| `resolve-next` reports `skipped` | An entry's `runIf` condition was false. | Nothing. The condition worked as intended. |
| `resolve-next` reports `waiting` | A test case is still `running`, with no `record-result` call. | Finish that test case's `record-result` call. |
| A step's hint has `confidence: "low"` | The recording captured a row position, not a business key. | This is expected. Add a test case input with `bindTo.kind: "grid-row-selector"` for the step. |
| `validate-plan` reports `entry declares suiteId "..." but ... has id "..."` | A plan entry's `suiteId` doesn't match the `id` in the suite file it points at. | Fix whichever value is wrong. They must match. |
| `plan-status` reports a suite as `missing` | No `testruns/<suiteId>-<runId>.json` exists for the plan run's ID. | Run `init-run` for that suite with the plan run's ID, and then run it. |
| `plan-status` reports `Plan run at ... belongs to plan "..."` | The plan run file was created for a different plan. | Pass the plan run file that was created for this plan. |
| `record-result` fails with a JSON syntax error in PowerShell | PowerShell changed the inline JSON. | Use the file options. See [Pass JSON through files in Windows PowerShell](#pass-json-through-files-in-windows-powershell). |

## Best practices

- **Never let the agent guess business data.** If a value should change from run to run, put it in
  a fixture.
- **Prefer OData outputs and expectations** whenever the result is a record you can query. They're
  much more reliable than reading text on a form.
- **Validate before every run.** `validate-testcase`, `validate-fixture`, `validate-suite` and
  `validate-plan` are free, and they catch mistakes before a live session spends time and tokens on
  them.
- **Review `example` values and literal test data before you commit them.** See
  [Data hygiene](#data-hygiene).
- **Keep one test case per business scenario**, and reuse it across suites and fixtures, instead of
  building one large test case that does everything.
- **Use a test plan** once you have several suites for one business area. It costs nothing extra
  per suite, and it gives you one combined result.

## Related content

- [Set up TRXA Studio for agent runs](SETUP.md)
- [Use TRXA Studio](STUDIO.md)
- [Use Model Context Protocol for finance and operations apps](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/copilot/copilot-mcp)
