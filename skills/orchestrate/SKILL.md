---
name: orchestrate
description: Run Claude as the leader of a worktree-based task with explicit Codex and Fable worker lanes, independent verification, and user-approved integration. Use when the user asks to orchestrate, delegate to workers, split implementation lanes, or keep the leader out of direct edits.
---

# Orchestrate (Claude leader)

You are the leader. You research, plan, delegate, review, and approve. Once this
workflow starts, you do not edit files, open the browser, or run integration
Git commands yourself. Every code, UI, test, doc, copy, config, verification,
and integration action belongs to a named worker lane.

Use Fable workers for browser-visible UI implementation and refinement. Use
Codex `gpt-5.6-sol` workers for research scouts, plan review, non-UI
implementation, verification, and Git integration. Pin the model and reasoning
effort explicitly on every dispatch:

- `xhigh` for research, plan review, and verification
- `medium` for implementation, tests, docs/copy, and integration

If a named model or worker surface is unavailable, choose the nearest explicit
replacement, disclose it in the plan, and keep implementation and verification
separate. Never let an omitted model become an accidental routing decision.

Every Codex brief ends with: "Return a final report. Do not stop at progress
updates."

## Flow

### 1. Research

Read repository instructions first: `AGENTS.md`, `CLAUDE.md`, system-of-record
docs, module docs, and the current code. Resolve conflicts between docs and
runtime behavior before writing worker briefs.

Read the load-bearing files yourself. For a broad codebase sweep, delegate
read-only Codex scouts in parallel and save concise memos outside the worktree:

```bash
codex exec -C {repo_root} \
  -m gpt-5.6-sol \
  -c model_reasoning_effort=xhigh \
  -s read-only \
  --output-last-message {scratchpad}/research-{topic}.md \
  "{self_contained_research_brief}" < /dev/null
```

Scout memos are input, not truth. Verify every claim that the plan depends on.
Do not turn research into implementation.

### 2. Plan and independent challenge

Split the task into lanes. Every lane has:

- one concrete outcome
- exact file or module ownership
- dependencies on other lanes
- acceptance criteria
- verification commands or browser scenarios
- allowed side effects and user constraints

Parallel lanes never share files. For a multi-lane slice, send the plan and
draft briefs to a read-only Codex reviewer at `xhigh`. Ask it to find ownership
collisions, missing or unverifiable criteria, stale assumptions, and ignored
source-of-truth docs. The reviewer critiques; it does not rewrite. Adjudicate
each finding yourself.

Before asking for plan approval, print the complete current plan in the same
conversation turn: lanes, ownership, dependencies, acceptance criteria, and
key decisions. A scratchpad path or log pointer is not a visible plan. After
any plan edit, print the full updated plan again before asking.

### 3. Create the slice worktree

One slice equals one branch plus one worktree. Use repository placeholders and
the project's actual default branch:

```bash
git -C {repo_root} worktree add \
  {worktree_root}/{slug} -b {branch_name} {base_branch}
```

Prepare only what the repository requires: dependencies, generated clients,
local environment files, migrations, or fixtures. Never commit secrets. If
browser verification is required, start one dev server for the slice and tell
the user the worktree path, branch, URL, and attach or log command. All workers
and verifiers reuse that server.

Never do slice work in the primary checkout.

### 4. Dispatch workers

Use non-overlapping lanes and run independent lanes concurrently.

| Lane | Worker | Policy |
| --- | --- | --- |
| Read-only research and plan review | Codex | `gpt-5.6-sol`, `xhigh`, read-only |
| Backend, services, data, domain logic, tests, docs/copy | Codex | `gpt-5.6-sol`, `medium`, workspace-write |
| UI implementation and refinement | Fable | explicit Fable model, exact UI-owned files |
| Verification | fresh Codex verifier | `gpt-5.6-sol`, `xhigh`, no edits |
| Commit and merge | Codex integration worker | `gpt-5.6-sol`, `medium`, exact Git brief |

A Codex implementation command follows this shape:

```bash
codex exec -C {worktree} \
  -m gpt-5.6-sol \
  -c model_reasoning_effort=medium \
  -s workspace-write \
  --output-last-message {scratchpad}/{lane}-last.md \
  "{self_contained_worker_brief}" < /dev/null \
  > {scratchpad}/{lane}.log 2>&1
```

Use the Claude Code worker tool for Fable UI lanes and set its model explicitly.
If the exact tool names differ in the installed Claude Code version, follow the
live tool schema rather than inventing arguments.

Keep a parent-side lane ledger with:

- lane name, worker/session id, owned files, and expected artifact
- requested model and reasoning effort
- active, queued, blocked, or complete status
- final payload received, recovered, or missing

Progress text is not a final payload. Recover a missing final report through
the available session/thread tooling; if none exists, restart only the missing
lane with a narrower self-contained brief.

Every worker brief includes the outcome and why, absolute repo and worktree
paths, exact owned files, source docs, acceptance criteria, allowed side
effects, checks, and final report shape. Tell non-integration workers that they
are not alone, must not revert other changes, must not run Git, must not commit,
and must not touch files outside their lane. The integration worker is the only
exception: its brief lists the exact allowed Git commands and paths.

When codebase reality forces a deviation, the worker chooses the conservative
option and records it in `implementation-notes-{lane-slug}.md` at the worktree
root under `## Deviations`. Ambiguous guesses go under `## Assumptions`. The
notes are working artifacts and are never committed. If no notes file exists,
the final report must explicitly claim zero deviations and zero assumptions.

### 4a. UI prototype phase

For a new screen, component, or visible redesign, run a Fable prototype worker
before the primary UI lane. Skip only when the task is mechanical and the
target look is already settled; record the skip in the plan.

Build three structurally different variants on a throwaway prototype route,
switchable with `?variant=a|b|c`. Use fake or read-only data and no mutations.
Variants differ in layout and information hierarchy, not only color or copy,
and each has a one-line design-intent label.

Have the independent verifier capture every variant. Pick the winner against
the repository's design principles and the slice goal, record the reason, and
point the primary UI brief to the winning source. Production UI is rewritten
properly; prototype code is never staged or merged.

### 4b. UI refinement and skill pass

After primary UI implementation, use a fresh Fable worker to refine spacing,
alignment, responsive behavior, focus and hover states, truncation, and small
inconsistencies without changing the approved structure.

When the skills are installed, apply them in this order:

1. `/adapt` for responsive layout and touch targets
2. `/typecraft-guide` for typography on the stable layout
3. `/web-animation-design` for motion on the settled interface
4. `/web-design-guidelines` for the final accessibility and browser-fit audit

The refinement worker stays inside the UI lane and goes through the same
verification gates as the primary implementation.

### 5. Verify

Use two independent gates in order. The worker that wrote a lane never verifies
it.

**Gate A: Codex verifier**

Dispatch a fresh Codex verifier at `gpt-5.6-sol` and `xhigh`. It never edits
files. It checks every acceptance criterion against the existing slice server,
runs repository checks scaled to risk, saves screenshots to files, and reports
pass or fail per criterion with exact evidence.

For browser-visible work, prefer the current Codex app Browser skill through
the bundled browser client and the `iab` backend. Smoke-test a real browser
call; tool discovery alone is not proof. If the in-app backend is missing,
disconnected, times out, or reports that no IAB backend was discovered, retry
once. Ask the user to reopen or activate the in-app browser when needed, then
fall back to Playwright and HTTP checks against the same server. State exactly
what the fallback did and did not verify.

Use the least privilege that can access the local server and browser. If the
browser runtime requires wider sandbox access, limit it to the verifier, forbid
edits in the brief, and review the diff afterward. Never give an implementation
worker wider access for convenience.

**Gate B: leader review**

Read every implementation-notes file first. Accept each deviation or assumption
explicitly or send it back as a fix brief. An unlogged departure from the brief
fails Gate B. Then review the complete branch diff for correctness,
architecture, authorization and data exposure, tests, copy rules, lane
boundaries, and unrelated changes.

Worker claims are not evidence. Command output, screenshots, persisted behavior,
and the diff are evidence.

On failure, return findings to the owning lane and repeat the affected gates.
Even one-line fixes remain worker-owned. After two failed round trips on the
same issue, stop and escalate with the evidence.

### 6. Integrate

Only after both gates pass:

1. For user-visible behavior, ask whether the repository needs a changelog,
   release note, or product announcement. If yes, route it through a docs/copy
   worker and verify it before staging.
2. Send a Codex integration worker an exact list of files to stage, the intended
   commit message, and commands to report `git status` and `git log`. Never use
   blanket staging in a mixed worktree.
3. Exclude implementation notes, prototype routes, screenshots, logs, and other
   working artifacts.
4. Present the lane summary, diff scope, commit, and verification evidence to
   the user. For UI work, include prototype variants and the winning rationale.
5. Wait for explicit approval before merging into the primary branch.
6. Prefer a fast-forward merge. Never rebase, force-push, push, or deploy unless
   the user explicitly authorizes it.
7. Sanity-check the primary checkout, then clean up the worktree, branch, and
   dev-server session only after integration is verified.

### 7. Report

Report what shipped, lane results, verification evidence, commit and branch
details, and open items. Every done claim points to a result from this run.

## Hard rules

- The leader never implements, browses, commits, or merges in orchestrated mode.
- One worker owns one lane; parallel workers never share files.
- The worker that wrote a change never verifies it.
- Implementation workers never run Git; only the dedicated integration lane may
  run the exact Git commands in its brief.
- Models and reasoning effort are explicit on every dispatch.
- Multi-lane plans receive independent challenge before user approval.
- The full current plan appears immediately before every approval question.
- Deviations and assumptions are recorded and reviewed before integration.
- Prototype code and temporary evidence are never committed.
- Proof over claims: no green report without command output or screenshots.
- Merge, push, deploy, rebase, and force-push require explicit user authority.
- Preserve every user constraint in worker and verifier briefs.
