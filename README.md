# TRXA-TOOL

Turn D365 Finance & Operations Task Recorder (`.axtr`) recordings into automated, data-driven
regression tests that an AI agent replays against a live or sandbox environment.

TRXA-TOOL does the deterministic work: it parses each recording into an ordered test script,
lets you define test cases, test data, suites and plans around it, and works out what runs next.
Your agent (Claude Code, Claude Cowork, Codex, or any agent that can drive MCP tools) does the
live part: it drives D365FO through the `d365-erp-mcp` server, checks each test's expectations,
and records pass/fail results you can review later.

## What you need

- **Node.js** 18.19 or later
- **An agent runtime that can call MCP tools**, such as [Claude Code](https://claude.com/claude-code),
  Claude Cowork, or [Codex](https://developers.openai.com/codex)
- **The `d365-erp-mcp` MCP server**, connected to the D365FO environment you want to test
- **One or more `.axtr` recordings**, recorded with D365FO's Task Recorder or exported from an LCS
  Business Process Modeler library

## Install

```sh
npm install -g trxa-tool
```

This installs the `trxa` command.

## Quick start

Run these from your own project folder:

```sh
trxa init-skill                  # once per project: installs the replay skill for Claude Code and Codex
trxa parse "recordings/My recording.axtr" -o testscripts/my-recording.testscript.json
trxa studio                      # optional: the browser UI for building test cases, suites and plans
```

Then ask your agent to run a suite, for example: *"use the replay-trxa-testplan skill to run
suites/my-suite.json"*.

Run `trxa help` for the full command list.

## Documentation

These files ship with the package (find them with `npm root -g`, under `trxa-tool/`):

- **[MANUAL.md](MANUAL.md)**: step-by-step setup and the full CLI workflow, from recording to
  test run.
- **[STUDIO.md](STUDIO.md)**: the browser UI.

## A note on data

A `.axtr` recording captures whatever was on screen when it was made, including real account
numbers, amounts and names. Test scripts and drafted test inputs carry that data forward. Review
anything derived from a real recording before sharing or committing it. See MANUAL.md's
"Data hygiene" section.

## Issues

Found a bug or a recording the tool can't handle? Open an issue at
https://github.com/elcanelo/trxa-tool/issues. Please don't attach recordings or test files that
contain real business data.

## License

Proprietary. Use is governed by the end-user license agreement in [LICENSE](LICENSE).
