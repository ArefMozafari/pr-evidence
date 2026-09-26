# pr-evidence as a cross-agent skill repository

Date: 2026-09-26

## Goal

Publish the `pr-evidence` skill so it installs in the widely used coding agents, not only in
Claude Code.

## Decision

One repository, laid out so that each agent's installer finds the same `SKILL.md`:

```
.claude-plugin/marketplace.json   Claude Code marketplace; Copilot CLI reads it too
.claude-plugin/plugin.json        Plugin manifest; the plugin root is the repository root
gemini-extension.json             Gemini CLI extension manifest
skills/pr-evidence/SKILL.md       The skill; `skills/*/SKILL.md` is what npx skills and gh skill discover
```

Every current major agent (Codex, Gemini CLI, Copilot, Cursor, Windsurf, OpenCode, Cline,
Goose, Amp, Claude Code) reads the Agent Skills `SKILL.md` format natively, so the skill body is
shared and only the thin manifests are agent-specific.

## Alternatives rejected

- **A bare root `SKILL.md`.** Loses the one-command plugin install in Claude Code and Copilot.
  A root `SKILL.md` would also shadow `skills/` for `npx skills`, so the two cannot coexist.
- **A Cursor plugin manifest (`plugin.json` at the root).** Cursor has no per-user plugin
  install; its users are served by `npx skills add` and a manual copy. Revisit if Cursor opens
  one.
- **A multi-skill collection repository.** Out of scope for sharing one skill; a later
  collection can list this repository as a marketplace source.

## Constraints

- The frontmatter holds only keys the Agent Skills spec allows. `user-invocable` was dropped:
  the reference validator and claude.ai upload reject unknown keys, and Claude Code treats
  skills as invocable by default.
- The skill text refers to no private document, and "emulator" is widened to a clean
  environment (emulator, simulator, or fresh browser profile) so it applies to web work too.
- `skills/pr-evidence/` must keep the folder name equal to the `name` field, which the spec
  requires.

## Verification

- CI runs the spec's reference validator and Claude Code's strict plugin validator on every
  push.
- Installs checked against the local checkout: Claude Code (`claude plugin marketplace add`
  and `install`, component inventory lists the skill), Gemini CLI (`extensions install`, then
  `skills list` shows it enabled), `npx skills add` (Codex and Cursor targets), and
  `gh skill install --from-local`.
- After the first push, all five README install commands were re-run against the GitHub
  repository, including Copilot CLI, which installs only from GitHub. Copilot's direct
  `plugin install owner/repo` form works but is deprecated, so the README uses its marketplace
  form.
