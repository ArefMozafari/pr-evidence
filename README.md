# pr-evidence

An agent skill that makes every pull request **show** its change instead of describing it.
A reviewer should grasp what moved without reading the diff.

It is a plain [Agent Skills](https://agentskills.io) `SKILL.md`, so it works in any coding
agent that reads the open format — Claude Code, Codex, Gemini CLI, GitHub Copilot, Cursor,
Windsurf, Cline, OpenCode, Goose, Amp, and others.

## What it teaches your agent

- **UI changes get before and after screenshots** — same screen, same device or viewport,
  same state — side by side in a two-column table.
- **Everything else gets one snippet** — the few lines that carry the change, not a tour of
  every file.
- **The "before" is captured first**, on the base branch at the start of the work, so nobody
  has to rebuild the old version afterwards.
- **Screenshots come from a clean environment** — an emulator, simulator, or fresh browser
  profile with a throwaway account — so no real account names, hostnames, or messages leak
  into a public repo.
- **Images live on a non-merging `pr-assets` branch**, each capture at a fresh path. GitHub
  has no API for PR image uploads, and its image cache keeps serving a file that was
  overwritten in place.
- **The PR body has six sections** — task summary, scope of work, testing instructions,
  screenshots, breaking changes, documentation/links — unless the repo has its own template.

The image-hosting rules are GitHub-specific; the rest applies to GitLab merge requests too.

## Example

The screenshots section of a PR written with this skill:

```markdown
## Screenshots

| Before | After |
| --- | --- |
| ![Before](https://github.com/acme/app/blob/pr-assets/42/1/settings-before.png?raw=true) | ![After](https://github.com/acme/app/blob/pr-assets/42/1/settings-after.png?raw=true) |
```

## Install

Pick the line for your agent.

| Agent | Command |
| --- | --- |
| Claude Code | `/plugin marketplace add ArefMozafari/pr-evidence`, then `/plugin install pr-evidence@pr-evidence` |
| Gemini CLI | `gemini extensions install https://github.com/ArefMozafari/pr-evidence` |
| GitHub Copilot CLI | `copilot plugin install ArefMozafari/pr-evidence` |
| Codex, Cursor, Windsurf, Cline, OpenCode, and more | `npx skills add ArefMozafari/pr-evidence` |
| Copilot and other agents, through the GitHub CLI | `gh skill install ArefMozafari/pr-evidence` |

### Manual install

Copy the `skills/pr-evidence` folder into your agent's skills directory:

| Agent | Directory |
| --- | --- |
| Claude Code | `~/.claude/skills/` |
| Codex, Gemini CLI, GitHub Copilot, Cursor, Windsurf, OpenCode, Goose, Amp | `~/.agents/skills/` |
| Cline | `~/.cline/skills/` |

## Development

Every push is checked by [`.github/workflows/validate.yml`](.github/workflows/validate.yml).
To run the same checks locally from the repository root:

```bash
uvx --from skills-ref==0.1.1 agentskills validate skills/pr-evidence
claude plugin validate --strict .claude-plugin/marketplace.json
claude plugin validate --strict .claude-plugin/plugin.json
```

## License

[MIT](LICENSE)
