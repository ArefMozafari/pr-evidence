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
.cursor-plugin/marketplace.json   Cursor marketplace, for the app's GitHub import and the Cursor CLI
.cursor-plugin/plugin.json        Cursor plugin manifest; the plugin root is the repository root
gemini-extension.json             Gemini CLI extension manifest
skills/pr-evidence/SKILL.md       The skill; `skills/*/SKILL.md` is what npx skills and gh skill discover
```

Every current major agent (Codex, Gemini CLI, Copilot, Cursor, Windsurf, OpenCode, Cline,
Goose, Amp, Claude Code) reads the Agent Skills `SKILL.md` format natively, so the skill body is
shared and only the thin manifests are agent-specific.

### Cursor

Cursor installs plugins from a GitHub repository per user: the app imports one as a marketplace
through Customize → From GitHub Repository, which requires `.cursor-plugin/marketplace.json`,
and the Cursor CLI adds one by git URL. The repository therefore ships a Cursor Plugin, the
format with its manifest in `.cursor-plugin/`, next to the Claude Code one.

- **The manifests are for the Cursor app.** The Cursor CLI already reads
  `.claude-plugin/marketplace.json`: before any Cursor file existed, `agent plugin marketplace
  add` indexed the plugin from `main`. The app's GitHub import indexed 0 plugins from the same
  commit.
- **The Cursor manifests repeat the Claude Code ones, except for `author` and `owner`.** Cursor's
  schema allows only `name` and `email` there and rejects `url`, so both carry the name alone.
- **`plugin.json` sets no `skills` field.** Cursor discovers `skills/*/SKILL.md` on its own, and
  an explicit path would replace that discovery rather than add to it.
- **CI checks both files against the JSON schemas Cursor publishes in `cursor/plugins`**, fetched
  at one pinned commit so an upstream change cannot break the build unannounced.

## Alternatives rejected

- **A bare root `SKILL.md`.** Loses the one-command plugin install in Claude Code and Copilot.
  A root `SKILL.md` would also shadow `skills/` for `npx skills`, so the two cannot coexist.
- **Cursor's Agent Plugin format, a `plugin.json` at the root.** Copilot CLI looks for a root
  `plugin.json` before `.claude-plugin/plugin.json`, so the root file would replace the manifest
  Copilot reads today, and the two copies would have to stay in sync. Cursor's GitHub import
  needs `.cursor-plugin/marketplace.json` either way.
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
