---
name: orchestrate
description: Run a task through a strict Claude leader and worker-lane flow in a dedicated worktree. Use when the user asks to orchestrate work, delegate to workers, split implementation lanes, verify independently, or keep Claude out of direct edits.
---

# Orchestrate

You are the leader. You research, plan, delegate, review, and approve. You do
not edit files in the primary checkout or in worker worktrees. Every code,
doc, copy, or config change goes to a worker lane, however small.

Your outputs are plans, worker briefs, reviews, and reports. Browser and
visual checks go to a verifier lane. Git commits and merges go to an
integration lane. The leader does not self-grade the work and does not merge
without explicit user approval.

Use this workflow when the task is substantial enough to benefit from lane
ownership, independent verification, or a clean worktree boundary. For tiny
one-file fixes, the orchestration overhead may be larger than the work.

## Flow

### 1. Research

Read the repository instructions first: `AGENTS.md`, `CLAUDE.md`, project docs,
and the current code relevant to the task. Resolve conflicts between docs and
runtime behavior before writing worker briefs.

Research deeply enough to produce:

- exact file or module ownership per lane
- acceptance criteria per lane
- known risks and edge cases
- verification commands
- user constraints such as read-only, no commit, no deploy, or no code

Do not turn research into implementation.

### 2. Plan

Split the task into lanes. Parallel lanes must not touch the same files. Each
lane needs a concrete outcome, owned files, acceptance criteria, and checks.

Get user approval before dispatching larger plans. For small orchestration
tasks, a compact plan summary is enough.

### 3. Worktree

One slice equals one branch plus one worktree. Prefer a sibling or configured
worktree root, not the primary checkout.

```bash
git worktree add {worktree_root}/{slug} -b {slug} {base_branch}
cd {worktree_root}/{slug}
```

Then prepare the worktree for the repository:

- install dependencies only if missing
- generate code clients if the project needs it
- copy local environment files only when needed, never commit secrets
- start one dev server for the slice if browser verification is required

Tell the user the worktree path, branch, local URL, and any `tmux attach`
command. Verifiers reuse the running server instead of starting another one.

Never do slice work directly in the primary checkout.

### 4. Dispatch Workers

Pick the worker by lane type. Every worker brief is self-contained and includes
the worktree path, exact owned files, source docs to read first, acceptance
criteria, verification commands, and report format.

Worker rules:

- no git commands
- no commits
- no edits outside owned files
- no starting unrelated work
- return a final report, not only progress updates

Recommended lane split:

| Lane | Worker | Owns |
| --- | --- | --- |
| Backend | Codex or another implementation worker | services, API routes, data models, domain logic, tests |
| UI build | Opus or strongest UI-capable worker | components, screens, styling, layout |
| UI refinement | Composer or detail-focused worker | spacing, alignment, states, truncation, finish |
| Verify | Codex verifier or independent reviewer | browser checks, functional checks, screenshots, command output |
| Integrate | Codex integration lane or trusted git operator | exact staged files, focused commit, approved merge |

If a worker is not available in the environment, replace it with the closest
equivalent and say so in the plan. Do not silently collapse writing and
verification into one lane.

### 5. UI Refinement

For browser-visible UI changes, run refinement before verification:

1. Detail pass: spacing, alignment, hover and focus states, truncation, mobile
   fit, and obvious polish.
2. Mandatory skill pass, in this order:
   - [`/adapt`](https://www.ui-skills.com/skills/pbakaus/adapt/) - responsive
     layout structure, breakpoints, fluid fit, and touch targets
   - [`/typecraft-guide-skill`](https://github.com/ehmo/typecraft-guide-skill/blob/main/skills/typecraft-guide/SKILL.md) -
     typography after the responsive layout is settled
   - [`/web-animation-design`](https://github.com/vercel-labs/open-agents/blob/main/.agents/skills/web-animation-design/SKILL.md) -
     motion and animation on the settled layout
   - [`/platform-web`](https://github.com/ehmo/platform-design-skills/blob/main/skills/web/SKILL.md) -
     final web guidelines audit, including contrast, focus states,
     reduced-motion behavior, and browser fit

`/platform-web` maps to the `web-design-guidelines` skill in
[ehmo/platform-design-skills](https://github.com/ehmo/platform-design-skills).

The refinement workers stay inside the UI lane files and keep the product's
existing design direction.

### 6. Verify

Two gates, in order.

**Gate A: independent verification**

The verifier did not write the implementation. It checks the affected routes or
commands against the acceptance criteria and reports pass or fail with evidence:

- screenshots for visual work
- command output for checks
- exact route, URL, viewport, or test command used
- any failed criterion with reproduction details

The verifier never edits code.

**Gate B: leader review**

Review the full diff yourself:

- correctness
- architecture and ownership boundaries
- authorization and data exposure
- tests and fixture impact
- copy and localization rules
- no unrelated refactors
- no worker lane overlap

Worker claims are not evidence. Command output, screenshots, and the diff are
evidence.

### 7. Fix Loop

If either gate fails, send findings back to the owning worker as a new
self-contained brief. Even one-line fixes go back through the lane boundary.

After two failed round trips on the same lane, stop and escalate with the exact
findings instead of looping indefinitely.

### 8. Integrate

Only after both gates pass:

1. Ask an integration lane to stage the exact files and make a focused commit.
2. Review the reported `git status` and `git log`.
3. Present the summary and verification evidence to the user.
4. Wait for explicit approval before merging into the primary branch.
5. Merge without rebasing or force pushing unless the user explicitly requests it.
6. Run a scaled sanity check in the primary checkout.
7. Remove the worktree and cleanup only after the merge is verified.

## Hard Rules

- The leader never edits files.
- One worker, one lane; no two workers own the same files at once.
- The worker that wrote the change never verifies it.
- Code workers do not run git.
- Integration happens only after verification passes.
- Merge only after explicit user approval.
- Proof over claims: no green report without command output or screenshots.
- Respect user constraints in every worker brief.
- Do not publish, deploy, push, or merge unless explicitly requested.

## Report Format

End with:

- what shipped
- lane summary
- verification evidence
- commit or branch details, if any
- open issues or blocked items

Every "done" claim must point to a real result from this run.
