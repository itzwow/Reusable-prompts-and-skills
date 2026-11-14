# Feature Factory

Build a "software factory" in this repo: a Claude Code skill, six subagents, two scripts and
markdown/JSON files that turn a feature description into a reviewed, human-approved branch.
Keep it small enough to understand in a few minutes. No new dependencies; plain bash,
markdown and one self-contained HTML file.

First, inspect the repo: language and framework, how the app is built and started, test
runner and how to run a subset, lint and typecheck commands, where features/modules live,
directory and naming conventions, commit conventions, CI. Record all of it in AGENTS.md
under "Factory". Agents read repo facts only from there — never re-discover them per job.

## Architecture

```
  /factory <feature>
        │
        ▼
  ┌──────────────┐  reads the request, the touched code and existing tests
  │ spec-writer  │──► spec.md: problem, scope, out of scope, acceptance criteria
  │   opus       │    (A1, A2…) each testable, open questions, risks
  └──────┬───────┘
         ▼
  ┌──────────────┐  reads spec.md + the real code
  │   planner    │──► plan.md: files to add/change, approach per file, data/API
  │   opus       │    changes, test plan (criterion → test), rollout/migration notes
  └──────┬───────┘
         ▼
  ┌──────────────┐  implements plan.md in the job's worktree
  │   builder    │◄─────────────────────────────────────────┐
  │   opus       │──► code + build.md (criterion → file/test map)│
  └──────┬───────┘                                          │
         ▼                                                  │
  ┌──────────────┐  bin/gate: build + tests + lint + tsc     │
  │  GATE script │  → round-N/gate.md                        │
  └──────┬───────┘  FAIL ───────────────────────────────────┤
         ▼ PASS                                             │
  ┌─────────────────────┐   ┌──────────────────┐            │
  │ reviewer-correctness│   │ reviewer-quality │ in parallel,│
  │   opus              │   │   sonnet         │ WAIT FOR ALL│
  └──────────┬──────────┘   └────────┬─────────┘            │
             └──────────┬────────────┘                      │
                        ▼  any CHANGES (max 2 resumes) ─────┘
                        ▼  both PASS
                 ┌──────────────┐
                 │   approver   │──► decision.md: APPROVE | ESCALATE
                 │   opus       │
                 └──────┬───────┘
                        ▼
            ready-for-human (always)  ──►  /factory accept <id>  → merge
                                      ──►  /factory rework <id> <note> → builder
```

Every job ends with a human. The factory never merges its own work.

## Files

```
.claude/skills/factory/SKILL.md          orchestrator; sole writer of job.json
.claude/agents/factory-spec-writer.md    opus
.claude/agents/factory-planner.md        opus
.claude/agents/factory-builder.md        opus
.claude/agents/factory-reviewer-correctness.md  opus
.claude/agents/factory-reviewer-quality.md      sonnet
.claude/agents/factory-approver.md       opus
factory/bin/gate                         build + tests + lint + tsc → round-N/gate.md
factory/bin/job                          only writer of job.json; refuses illegal stage moves
factory/backlog.md                       "- [ ] <feature>" lines for /factory next
factory/dashboard.html                   live board
factory/jobs/<job-id>/                   gitignored runtime state:
  job.json, spec.md, plan.md, build.md,
  round-<N>/gate.md, round-<N>/review-correctness.md, round-<N>/review-quality.md,
  rework-<k>.md, decision.md
```

## Contracts (the orchestrator reads only first lines)

- Agents read context from files, write exactly one file (the builder also writes code),
  and reply with one line. Every file has a `## Summary` of 3–5 bullets after line 1.
- spec.md line 1: `SPEC: READY` | `SPEC: BLOCKED`
- plan.md line 1: `PLAN: READY` | `PLAN: BLOCKED`
- build.md line 1: `STATUS: READY` | `STATUS: BLOCKED`
- gate.md line 1: `GATE: PASS` | `GATE: FAIL`
- reviews line 1: `VERDICT: PASS` | `VERDICT: CHANGES`
- decision.md line 1: `APPROVE` | `ESCALATE`

## Agent rules

- factory-spec-writer: no Edit tool. Acceptance criteria are observable and testable, each
  with an id. Anything it had to assume goes under "Assumptions"; anything it cannot decide
  goes under "Open questions" and makes the spec BLOCKED only if the work can't start.
- factory-planner: no Edit tool. Plans against the code as it is, reusing existing modules
  and patterns; names every file it would touch and why. Flags any plan step that needs a
  product decision or a breaking change.
- factory-builder: implements plan.md and nothing else — no drive-by refactors, no new
  dependencies without saying so in build.md. Follows AGENTS.md conventions. Adds a test per
  acceptance criterion. Runs the gate's commands itself until green before finishing.
  Commits on feat/<job-id> in small, labelled commits.
- factory-reviewer-correctness: no Edit tool. Every acceptance criterion is actually
  implemented and actually proven by a test that would fail without the change. Hunts real
  defects: edge cases, error paths, concurrency, data loss, security, migrations.
  Each finding: file:line, how it fails, suggested fix.
- factory-reviewer-quality: no Edit tool. Repo conventions, naming, dead code, duplication,
  over-engineering, test quality, no weakened or skipped tests without a stated reason.
  Each finding: file:line, problem, fix.
- factory-approver: no Edit tool. APPROVE only if the gate passed in the final round, both
  reviews PASS, and it re-ran the test suite subset for this feature itself. ESCALATE if the
  spec rests on a product assumption, the change is breaking or touches data/migrations/auth,
  or any criterion is unimplemented. decision.md ends with "For the human": what to check,
  which files, how to try it.

## Orchestrator (SKILL.md)

- Job id: next 3-digit number + feature slug, claimed with mkdir (no -p).
- Worktree per job: `git worktree add -b feat/<job-id> ../<repo>-factory/<job-id> main`;
  job state stays in the main checkout. Each job gets its own app port (5100 + 10n) so
  several features run in parallel.
- Spawn the builder named `factory-builder-<job-id>`; resume it with SendMessage after a
  failed gate or CHANGES, pointing at the round folder. MAX_RESUMES = 2, then ESCALATE.
- Spawn both reviewers in one message; WAIT FOR ALL; write each verdict to job.json as it lands.
- stages: spec, plan, build, gate, review, approve, ready-for-human, accepted, needs-human.
  Every job ends in ready-for-human or needs-human. Never merge without /factory accept.
- Post one line per transition, e.g. `003-csv-export: round 1 gate PASS, correctness CHANGES (A4 untested)`.

Commands:

```
/factory <feature>               new job
/factory next                    next unchecked feature in backlog.md
/factory accept <job-id>         human approval: merge feat/<job-id> into main (main must be clean)
/factory rework <job-id> <note>  note → rework-<k>.md, fresh named builder, continue from gate
```

## Dashboard (factory/dashboard.html)

One self-contained file, no build step, served with `python3 -m http.server` from factory/,
polling jobs/ every 2 s. A column per stage, a card per job: id, feature, current stage,
age, and per round a row of chips: gate, correctness, quality. A criteria bar on each card
(criteria implemented / criteria in spec). ready-for-human and needs-human stand out.
Clicking a card shows every file for that job, rendered as markdown.

## Finish

Commit on main (factory/jobs/ ignored), add two real features from this repo's TODOs to
backlog.md, print how to run /factory, the dashboard and parallel jobs. Don't run a job yourself.
