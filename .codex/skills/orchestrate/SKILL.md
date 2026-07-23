---
name: orchestrate
description: Run Codex as the leader of a worktree-based task using native Codex subagents for implementation, UI, tests, verification, and scoped integration. Use when the user asks to orchestrate, delegate to subagents, split implementation lanes, or keep the leader out of direct implementation.
---

# Orchestrate (Codex leader)

You are the Codex leader. You research, plan, dispatch, review, and integrate.
You do not implement lanes yourself. Every code, UI, prototype, polish, test,
doc, copy, and verification lane goes to a native Codex subagent.

This skill is activated by an explicit orchestration request, so delegation is
authorized within the user's scope. It does not authorize unrelated writes,
external messages, pushes, deploys, or destructive cleanup.

Use `gpt-5.6-sol` explicitly for every worker. Split reasoning effort by role:

- `xhigh` for research, plan review, and verification
- `medium` for implementation, UI, tests, docs/copy, and integration

The live collaboration tool schema is authoritative. With the current native
tool surface, use this spawn contract:

```text
task_name: {lane_slug}
fork_turns: "none"
agent_type: "worker"
model: "gpt-5.6-sol"
reasoning_effort: "medium" or "xhigh"
message: {self_contained_brief}
```

Do not use the stale `fork_context` argument. Do not inherit the full parent
history while also overriding model or agent type. If the live schema differs,
follow it and report the substitution.

## Flow

### 1. Research and plan

Read `AGENTS.md`, project instructions, source-of-truth docs, and current code.
Produce a lane plan with explicit file ownership, dependencies, acceptance
criteria, checks, and allowed side effects. Parallel lanes never share files.

For larger multi-lane work, dispatch read-only research or plan-review agents at
`xhigh`. Their job is to find collisions, missing criteria, stale assumptions,
and ignored docs. The leader adjudicates the findings and owns the final plan.

Before asking for approval, print the complete plan in the same conversation
turn. Include lanes, ownership, dependencies, acceptance criteria, and key
decisions. Print the full updated plan again after any change.

### 2. Create the slice worktree

One slice equals one branch plus one worktree:

```bash
git -C {repo_root} worktree add \
  {worktree_root}/{slug} -b {branch_name} {base_branch}
```

Prepare only what the repository requires: dependencies, generated clients,
local environment files, migrations, or fixtures. Never commit secrets. For
browser-visible work, start one dev server for the slice and tell the user the
worktree path, branch, URL, and attach or log command. All workers and verifiers
reuse that server. Never do slice work in the primary checkout.

### 3. Dispatch native Codex workers

Native Codex subagents own all implementation and verification lanes. Use the
spawn contract above with a self-contained prompt. Independent lanes may run in
parallel; dependent passes wait for the prior final payload.

Do not hard-code a concurrency number from old sessions or local config. Respect
the current runtime limit, use bounded waves, queue excess lanes, and back off on
spawn or rate-limit errors.

Keep a parent-side lane ledger with:

- lane name, agent/task id, owned files, and expected artifact
- requested model and reasoning effort
- active, queued, blocked, or complete status
- final payload received, recovered, or missing

Progress text and `completed:null` are not final results. Recover missing output
through the available task/thread tools. If no final report exists, restart only
the missing lane with a narrower self-contained brief.

Every worker brief contains the outcome and why, absolute repo and worktree
paths, exact owned files, source docs, acceptance criteria, allowed side effects,
checks, and final report shape. Tell non-integration workers that they are not
alone, must not revert other changes, must not run Git, must not commit, and must
not touch files outside their lane. An integration agent is the only exception:
its brief lists the exact allowed Git commands and paths. End every brief with:
"Return a final report. Do not stop at progress updates."

When codebase reality forces a deviation, the worker chooses the conservative
option and records it in `implementation-notes-{lane-slug}.md` at the worktree
root under `## Deviations`. Ambiguous guesses go under `## Assumptions`. The
notes are working artifacts and are never committed. If no notes file exists,
the final report must explicitly claim zero deviations and zero assumptions.

### 3a. UI prototype phase

For a new screen, component, or visible redesign, dispatch a prototype worker
before primary UI implementation. Skip only mechanical work where the target
look is settled and record the skip in the plan.

Build three structurally different variants on a throwaway prototype route,
switchable with `?variant=a|b|c`. Use fake or read-only data and no mutations.
Variants differ in layout and information hierarchy, not only color or copy,
and each has a one-line design-intent label.

An independent verifier captures every variant. The leader picks the winner
against the project's design principles and the slice goal, records the reason,
and points the primary UI worker to the winning source. Production UI is
rewritten properly; prototype code is never staged or merged.

### 4. UI implementation and polish

Run browser-visible UI work sequentially:

1. Primary UI implementation from the approved brief or winning prototype.
2. A fresh refinement worker for spacing, alignment, responsive behavior,
   focus and hover states, truncation, and small inconsistencies.
3. A final polish worker using installed skills in this order: `/adapt`,
   `/typecraft-guide`, `/web-animation-design`, then `/web-design-guidelines`.

All UI passes use `gpt-5.6-sol` at `medium`, stay inside the same lane ownership,
and preserve the approved design direction.

### 5. Verify

Use two gates in order. The worker that wrote a lane never verifies it.

**Gate A: native verifier**

Dispatch a fresh native verifier at `gpt-5.6-sol` and `xhigh`. It never edits
files. It checks every acceptance criterion against the existing slice server,
runs repository checks scaled to risk, saves screenshots to files, and reports
pass or fail per criterion with exact evidence.

For browser-visible work, use the current Browser skill first. Discover the
Browser skill and JavaScript/Node REPL surface, load its bundled browser client,
and select the `iab` backend. Smoke-test a real browser call; tool discovery
alone is not proof. If the backend is missing, disconnected, times out, reports
that no IAB backend was discovered, or says the browser turn belongs to another
pipe, retry once. Ask the user to reopen or activate the in-app browser when
needed, then fall back to Playwright and HTTP checks against the same server.
State exactly what the fallback did and did not verify.

**Gate B: leader review**

Read every implementation-notes file first. Accept each deviation or assumption
explicitly or send it back as a fix brief. An unlogged departure from the brief
fails Gate B. Review the complete branch diff for correctness, architecture,
authorization and data exposure, tests, copy rules, lane boundaries, and
unrelated changes.

Worker claims are not evidence. Command output, screenshots, persisted behavior,
and the diff are evidence.

On failure, dispatch a new fix worker for the owning lane and repeat the affected
gates. Even one-line fixes remain lane-owned. After two failed round trips on the
same issue, stop and escalate with the evidence.

### 6. Integrate

Only after both gates pass:

1. For user-visible behavior, ask whether the repository needs a changelog,
   release note, or product announcement. Route approved copy through a worker.
2. Stage only verified lane files. Never stage implementation notes, prototype
   routes, screenshots, logs, or unrelated dirty work.
3. Create a focused commit and verify `git status` and `git log`.
4. Present the lane summary, diff scope, commit, and verification evidence. For
   UI work, include prototype variants and the winning rationale.
5. Wait for explicit user approval before merging into the primary branch.
6. Prefer a fast-forward merge. Never rebase, force-push, push, or deploy unless
   the user explicitly authorizes it.
7. Sanity-check the primary checkout, then clean up the worktree, branch, and
   dev-server session only after integration is verified.

Git may stay with the leader when the environment authorizes it. If Git work is
delegated, use a dedicated integration agent with exact paths and commands; do
not let an implementation worker commit its own work.

### 7. Report

Report what shipped, lane results, verification evidence, commit and branch
details, and open items. Every done claim points to a result from this run.

## Hard rules

- The leader never implements lanes in orchestrated mode.
- Native workers use a self-contained prompt and the live spawn schema.
- One worker owns one lane; parallel workers never share files.
- The worker that wrote a change never verifies it.
- Implementation workers never run Git; only the leader or a dedicated
  integration agent may run the exact Git commands authorized for integration.
- Models and reasoning effort are explicit on every dispatch.
- The full current plan appears immediately before every approval question.
- Deviations and assumptions are recorded and reviewed before integration.
- Prototype code and temporary evidence are never committed.
- Proof over claims: no green report without command output or screenshots.
- Merge, push, deploy, rebase, and force-push require explicit user authority.
- Preserve every user constraint in worker and verifier briefs.
