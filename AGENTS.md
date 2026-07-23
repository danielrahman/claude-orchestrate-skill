# claude-orchestrate-skill

This repository packages public orchestration skills for Claude Code and
Codex. There is no build step. Keep changes text-focused and easy to audit.

## Map

- `skills/orchestrate/SKILL.md` - Claude-led skill and Claude Code plugin body
- `.codex/skills/orchestrate/SKILL.md` - native Codex-led skill
- `.claude-plugin/` - Claude Code plugin and marketplace metadata
- `docs/*-orchestrator-readme.webp` - generated README flow cards
- `docs/orchestrate-twitter-card.png` - original Claude-led social preview

## Working agreements

- Keep both skills portable: no private paths, project names, reserved ports,
  secrets, or personal-machine assumptions.
- Preserve the leader, worker, verifier, and integration boundaries.
- Never make worker self-verification acceptable.
- Use current tool argument names and placeholders instead of stale local
  examples.
- Keep platform-specific behavior in the matching skill; do not pretend Claude
  Code and native Codex workers share one dispatch API.
- If shared behavior changes, update both skills and `README.md` where relevant.
- Keep the existing Claude plugin path stable for current installations.
