# E2E UI Test Factory

Build an "E2E test factory" in this repo: a Claude Code skill, six subagents, two scripts and
markdown/JSON files that turn a widget name into reviewed Playwright e2e tests. Keep it small
enough to understand in a few minutes. Model it on the software factory already in this repo
(factory/ and .claude/skills/factory/) and reuse its patterns, dashboard and worktree logic,
but keep it separate (e2e-factory/, /e2e).

First, inspect the repo: where widgets live and how they're registered/rendered, how to start
the app on a given port, auth/test users, existing Playwright config, fixtures, page objects,
helpers and conventions. Record all of it in AGENTS.md under "E2E testing". Agents read
repo facts only from there.

If Playwright Test Agents aren't set up, run `npx playwright init-agents --loop=claude`
and keep the generated .mcp.json, seed test and agent files. Our agents use that MCP
server's browser and test tools for live exploration and selector checks; look up the
exact tool names after init. If my existing Playwright skill is present in .claude/skills/,
the test-writer and reviewer-quality agents must load and follow it.

## Architecture

```
  /e2e <widget>
        │
        ▼
  ┌──────────────┐  reads widget code, props, state, API calls, existing tests
  │ spec-writer  │──► spec.md: behaviour, states, scenario matrix (happy, edge,
  │   opus       │    error, empty, loading, permissions, a11y, responsive), each
  └──────┬───────┘    scenario with an id (S1, S2…) and expected result
         ▼
  ┌──────────────┐  starts app on its port, drives the real widget through every
  │   analyser   │  scenario via Playwright MCP, records actual behaviour,
  │   sonnet     │──► analysis.md (+ artifacts/ screenshots, network calls to mock,
  └──────┬───────┘    stable locators, flake risks, SPEC vs REALITY mismatches)
         ▼
  ┌──────────────┐  writes tests (fixtures/page objects per AGENTS.md), runs
  │ test-writer  │◄─────────────────────────────────────────┐
  │   opus       │──► test files + build.md (scenario → test map)│
  └──────┬───────┘                                          │
         ▼                                                  │
  ┌──────────────┐  bin/gate: tests --repeat-each=3          │
  │  GATE script │  --retries=0, + lint + tsc → round-N/gate.md
  └──────┬───────┘  FAIL ───────────────────────────────────┤
         ▼ PASS                                             │
  ┌──────────────────┐   ┌──────────────────┐               │
  │ reviewer-coverage│   │ reviewer-quality │  in parallel,  │
  │   opus           │   │   sonnet         │  WAIT FOR ALL  │
  └────────┬─────────┘   └────────┬─────────┘               │
           └──────────┬───────────┘                         │
                      ▼  any CHANGES (max 2 resumes) ───────┘
                      ▼  both PASS
               ┌──────────────┐
               │   approver   │──► decision.md: APPROVE | ESCALATE
               │   opus       │
               └──────┬───────┘
                      ▼
            ready-for-human (always)  ──►  /e2e accept <id>  → merge
                                      ──►  /e2e rework <id> <note> → test-writer
```

Every job ends with a human. The factory never merges tests on its own.

## Files

```
.claude/skills/e2e/SKILL.md           orchestrator; sole writer of job.json
.claude/agents/e2e-spec-writer.md     opus
.claude/agents/e2e-analyser.md        sonnet
.claude/agents/e2e-test-writer.md     opus
.claude/agents/e2e-reviewer-coverage.md  opus
.claude/agents/e2e-reviewer-quality.md   sonnet
.claude/agents/e2e-approver.md        opus
e2e-factory/bin/gate                  runs the job's tests + lint + tsc → round-N/gate.md
e2e-factory/bin/job                   only writer of job.json; refuses illegal stage moves
e2e-factory/backlog.md                "- [ ] <widget>" lines for /e2e next
e2e-factory/dashboard.html            live board
e2e-factory/jobs/<job-id>/            gitignored runtime state:
  job.json, spec.md, analysis.md, artifacts/, build.md,
  round-<N>/gate.md, round-<N>/review-coverage.md, round-<N>/review-quality.md,
  rework-<k>.md, decision.md
```

## Contracts (the orchestrator reads only first lines)

- Agents read context from files, write exactly one file (test-writer also writes tests),
  and reply with one line. Every file has a `## Summary` of 3–5 bullets after line 1.
- spec.md line 1: `SPEC: READY` | `SPEC: BLOCKED`
- analysis.md line 1: `ANALYSIS: READY` | `ANALYSIS: BLOCKED` (app won't start, widget unreachable)
- build.md line 1: `STATUS: READY` | `STATUS: BLOCKED`
- gate.md line 1: `GATE: PASS` | `GATE: FAIL`
- reviews line 1: `VERDICT: PASS` | `VERDICT: CHANGES`
- decision.md line 1: `APPROVE` | `ESCALATE`

## Agent rules

- e2e-spec-writer: reads the widget's code and its callers; never runs the browser.
  Scenario matrix as a table: id, scenario, preconditions, steps, expected result, priority.
- e2e-analyser: no Edit tool, never writes tests. Runs each scenario live, records what
  actually happened, the best locator per element (getByRole / getByLabel / getByTestId,
  in that order), network requests to mock or wait on, timing/animation flake risks.
  Lists every SPEC vs REALITY mismatch: these are possible product bugs.
- e2e-test-writer: one spec file per widget under the e2e folder from AGENTS.md; reuses
  existing fixtures and page objects, adds one only if needed. Each test is titled with its
  scenario id ("S3: shows empty state when API returns []"). User-facing locators, web-first
  assertions (expect(...).toBeVisible/ toHaveText…), no waitForTimeout, no test.only, each
  test independent, deterministic data via route mocking or fixtures. A confirmed product bug
  becomes `test.fixme` with the bug described, never a test that asserts the buggy behaviour.
  Runs its own tests until green before finishing. Commits on e2e/<job-id>.
- e2e-reviewer-coverage: no Edit tool. Every spec scenario maps to a test whose assertions
  actually prove the expected result (not just that something is visible). Flags missing
  negative/error/edge cases and every analyser mismatch that went unhandled.
- e2e-reviewer-quality: no Edit tool. Playwright best practices and the repo's conventions:
  locator stability, assertion strength, isolation, flake risk, duplication, naming,
  no weakened or skipped tests without a reason. Each finding: file:line, problem, fix.
- e2e-approver: no Edit tool. APPROVE only if the gate passed in the final round, both
  reviews PASS, and it re-ran one test itself. ESCALATE if any test.fixme exists (possible
  bug for a human), the spec rests on a product assumption, or tests need app-code changes
  (new test ids, etc.). decision.md ends with "For the human": what to check, which files.

## Orchestrator (SKILL.md)

- Job id: next 3-digit number + widget slug, claimed with mkdir (no -p).
- Worktree per job: `git worktree add -b e2e/<job-id> ../<repo>-e2e/<job-id> main`;
  job state stays in the main checkout. Each job gets its own app port (6100 + 10n) and its own
  Playwright output dir and browser session, so several widgets run in parallel.
- Spawn the test-writer named `e2e-writer-<job-id>`; resume it with SendMessage after a failed
  gate or CHANGES, pointing at the round folder. MAX_RESUMES = 2, then ESCALATE.
- Spawn both reviewers in one message; WAIT FOR ALL; write each verdict to job.json as it lands.
- stages: spec, analyse, write, gate, review, approve, ready-for-human, accepted, needs-human.
  Every job ends in ready-for-human or needs-human. Never merge without /e2e accept.
- Post one line per transition, e.g. `004-date-picker: round 1 gate PASS, coverage CHANGES (S7, S9 untested)`.

Commands:

```
/e2e <widget>                 new job
/e2e next                     next unchecked widget in backlog.md
/e2e accept <job-id>          human approval: merge e2e/<job-id> into main (main must be clean)
/e2e rework <job-id> <note>   note → rework-<k>.md, fresh named test-writer, continue from gate
```

## Dashboard (e2e-factory/dashboard.html)

Same approach as factory/dashboard.html: one self-contained file, `python3 -m http.server`
from e2e-factory/, polls jobs/ every 2 s. Columns per stage; per round a row of chips:
gate, coverage, quality. A scenario coverage bar on each card (tests written / scenarios
in spec). ready-for-human and needs-human stand out. Clicking a card shows every file,
analyser screenshots inline, and the scenario → test table from build.md.

## Finish

Commit on main (jobs/ ignored), add two real widget names from this repo to backlog.md,
print how to run /e2e, the dashboard and parallel jobs. Don't run a job yourself.
