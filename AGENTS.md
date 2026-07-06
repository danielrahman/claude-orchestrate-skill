# claude-orchestrate-skill

This repository is a small Claude Code skill/plugin package. There is no build
step. Keep changes simple, text-focused, and easy to audit.

## Map

- `skills/orchestrate/SKILL.md` - the public skill body
- `.claude-plugin/` - plugin and marketplace metadata
- `docs/orchestrate-twitter-card.png` - social preview image

## Working Agreements

- Keep the skill generic: no private paths, repo-specific project names, local
  ports, secrets, or personal machine assumptions.
- Keep the leader/worker boundary explicit.
- Do not make worker self-verification acceptable.
- Use current command shapes and placeholders rather than stale local examples.
- If behavior changes, update both `README.md` and the public skill body.
