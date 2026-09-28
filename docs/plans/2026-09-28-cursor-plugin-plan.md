# Cursor plugin manifest implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let Cursor users install the `pr-evidence` skill as a plugin straight from this GitHub repository (issue #1).

**Architecture:** Two new files in `.cursor-plugin/`, the Cursor Plugin format, mirror the existing `.claude-plugin/` files. CI checks them against the JSON schemas Cursor publishes, and the README gains a Cursor install row. The design is the "Cursor" section of `docs/plans/2026-09-26-pr-evidence-repo-design.md`.

**Tech Stack:** JSON manifests, GitHub Actions, `ajv-cli` 5.0.0 with `ajv-formats` 3.0.1, the Cursor CLI (`agent`), Claude Code, Gemini CLI 0.61.0, Copilot CLI 1.0.88.

## Global Constraints

- Branch `feature/cursor-plugin-manifest`, cut from `origin/main` at `c8c085e`.
- No commit, push, or pull request without the owner's explicit approval; the plan stops before each.
- `author` and `owner` carry `name` only: Cursor's schema rejects `url`, and no email goes into the repository.
- `.cursor-plugin/plugin.json` sets no `skills` field.
- Cursor's schemas are fetched from `cursor/plugins` at commit `ecc249f1e306fc64ddf83c7bed16cacf7c2239db`.
- Agent installs used for verification run against a throwaway home or config folder in the scratch directory, never the owner's real agent configuration. Cursor keeps marketplaces on the owner's account, so each Cursor test marketplace is removed right after its check.
- Commit subjects: `feat: add a Cursor plugin manifest and marketplace`, `ci: validate the Cursor manifests against Cursor's schemas`, `docs: add the Cursor plugin install to the README`, `docs: record the Cursor plugin design and plan`, and, once the GitHub installs have run, `docs: record the Cursor plugin install checks`.

In the commands below, `$SCRATCH` stands for the session's scratch directory and `$WORKTREE` for this worktree's absolute path. Write both out literally when running a command.

---

### Task 1: Cursor manifests

**Files:**
- Create: `.cursor-plugin/plugin.json`
- Create: `.cursor-plugin/marketplace.json`

**Interfaces:**
- Produces: the two manifest paths that Tasks 2 to 5 validate and install.

- [ ] **Step 1: Fetch the pinned schemas into the scratch directory**

```bash
curl -fsSL -o $SCRATCH/cursor-plugin.schema.json https://raw.githubusercontent.com/cursor/plugins/ecc249f1e306fc64ddf83c7bed16cacf7c2239db/schemas/plugin.schema.json
curl -fsSL -o $SCRATCH/cursor-marketplace.schema.json https://raw.githubusercontent.com/cursor/plugins/ecc249f1e306fc64ddf83c7bed16cacf7c2239db/schemas/marketplace.schema.json
```

- [ ] **Step 2: Show that the check catches the Claude manifest's `author.url`**

```bash
npx -y -p ajv-cli@5.0.0 -p ajv-formats@3.0.1 ajv validate --spec=draft7 -c ajv-formats -s $SCRATCH/cursor-plugin.schema.json -d .claude-plugin/plugin.json
```

Expected: exit code 1, `must NOT have additional properties` with `additionalProperty: 'url'` at `/author`.

- [ ] **Step 3: Run the check on the missing Cursor manifest**

```bash
npx -y -p ajv-cli@5.0.0 -p ajv-formats@3.0.1 ajv validate --spec=draft7 -c ajv-formats -s $SCRATCH/cursor-plugin.schema.json -d .cursor-plugin/plugin.json
```

Expected: exit code 2, `Cannot find data file .cursor-plugin/plugin.json`.

- [ ] **Step 4: Create `.cursor-plugin/plugin.json`**

```json
{
  "name": "pr-evidence",
  "version": "1.0.0",
  "description": "Before/after screenshots for UI changes, one snippet for everything else, images hosted on a non-merging pr-assets branch, and a six-section PR body.",
  "author": {
    "name": "Aref Mozafari"
  },
  "homepage": "https://github.com/ArefMozafari/pr-evidence",
  "repository": "https://github.com/ArefMozafari/pr-evidence",
  "license": "MIT",
  "keywords": ["pull-request", "code-review", "screenshots", "github", "agent-skill"]
}
```

- [ ] **Step 5: Create `.cursor-plugin/marketplace.json`**

```json
{
  "name": "pr-evidence",
  "owner": {
    "name": "Aref Mozafari"
  },
  "metadata": {
    "description": "Agent skill that makes every pull request show its change instead of describing it."
  },
  "plugins": [
    {
      "name": "pr-evidence",
      "source": "./",
      "description": "Before/after screenshots for UI changes, one snippet for everything else, images hosted on a non-merging pr-assets branch, and a six-section PR body."
    }
  ]
}
```

- [ ] **Step 6: Run both checks**

```bash
npx -y -p ajv-cli@5.0.0 -p ajv-formats@3.0.1 ajv validate --spec=draft7 -c ajv-formats -s $SCRATCH/cursor-plugin.schema.json -d .cursor-plugin/plugin.json
npx -y -p ajv-cli@5.0.0 -p ajv-formats@3.0.1 ajv validate --spec=draft7 -c ajv-formats -s $SCRATCH/cursor-marketplace.schema.json -d .cursor-plugin/marketplace.json
```

Expected: `.cursor-plugin/plugin.json valid` and `.cursor-plugin/marketplace.json valid`.

### Task 2: CI check

**Files:**
- Modify: `.github/workflows/validate.yml`, a new last step after the Gemini step

**Interfaces:**
- Consumes: the two manifest paths from Task 1.

- [ ] **Step 1: Add the step**

```yaml
      # Cursor's schemas are pinned to one commit of cursor/plugins, so an upstream change cannot
      # break this build unannounced.
      - name: Validate the Cursor plugin and marketplace manifests
        env:
          CURSOR_SCHEMAS_URL: https://raw.githubusercontent.com/cursor/plugins/ecc249f1e306fc64ddf83c7bed16cacf7c2239db/schemas
        run: |
          curl -fsSL -o "$RUNNER_TEMP/plugin.schema.json" "$CURSOR_SCHEMAS_URL/plugin.schema.json"
          curl -fsSL -o "$RUNNER_TEMP/marketplace.schema.json" "$CURSOR_SCHEMAS_URL/marketplace.schema.json"
          npx --yes -p ajv-cli@5.0.0 -p ajv-formats@3.0.1 ajv validate --spec=draft7 -c ajv-formats -s "$RUNNER_TEMP/plugin.schema.json" -d .cursor-plugin/plugin.json
          npx --yes -p ajv-cli@5.0.0 -p ajv-formats@3.0.1 ajv validate --spec=draft7 -c ajv-formats -s "$RUNNER_TEMP/marketplace.schema.json" -d .cursor-plugin/marketplace.json
```

- [ ] **Step 2: Run the step's commands locally with `RUNNER_TEMP` set to the scratch directory**

Save the four `run` lines to `$SCRATCH/cursor-step.sh`, then run from the worktree root:

```bash
RUNNER_TEMP=$SCRATCH CURSOR_SCHEMAS_URL=https://raw.githubusercontent.com/cursor/plugins/ecc249f1e306fc64ddf83c7bed16cacf7c2239db/schemas bash $SCRATCH/cursor-step.sh
```

Expected: both files report `valid`, exit code 0.

- [ ] **Step 3: Run the rest of the workflow locally**

```bash
uvx --from skills-ref==0.1.1 agentskills validate skills/pr-evidence
npx --yes @anthropic-ai/claude-code@2 plugin validate --strict .claude-plugin/marketplace.json
npx --yes @anthropic-ai/claude-code@2 plugin validate --strict .claude-plugin/plugin.json
jq -e '.name == "pr-evidence" and (.version | type == "string") and (.description | type == "string")' gemini-extension.json
```

Expected: `Valid skill: skills/pr-evidence`, `✔ Validation passed` twice, `true`.

### Task 3: Real installs from the local checkout

**Files:** none changed.

**Interfaces:**
- Produces: the exact Cursor CLI install command that Task 4 writes into the README.

- [ ] **Step 1: Ask the owner before downloading the Cursor CLI.** The owner runs Cursor's documented script, since the worktree guard refuses piping into `bash`:

```bash
curl https://cursor.com/install -fsS | bash
```

Expected: `agent --version` prints a version.

- [ ] **Step 2: Read the plugin commands**

```bash
agent plugin --help
agent plugin marketplace add --help
```

Observed with `2026.09.26-dd393fe`: the shell only manages marketplaces (`add`, `list`, `remove`, `update`; `add` takes `--git-ref`). Plugins install inside a session, from `/plugins`. Every plugin command needs the owner's sign-in (`agent login`).

- [ ] **Step 3: Find out what each route reads**

`--plugin-dir` loads any folder with a `skills/` directory, manifest or not, so it proves nothing about the manifests. The marketplace routes do: on `main` at `c8c085e`, before any Cursor file existed, `agent plugin marketplace add https://github.com/ArefMozafari/pr-evidence` indexed 1 plugin, so the CLI reads `.claude-plugin/marketplace.json`. The owner's import of the same repository in the app's Customize, From GitHub Repository, showed no plugin. The Cursor manifests were therefore expected to serve the app; after the merge the app still showed no plugin, which is tracked in #9. Remove each test marketplace with `agent plugin marketplace remove <name>`.

- [ ] **Step 4: Claude Code still installs, in a throwaway config folder**

```bash
CLAUDE_CONFIG_DIR=$SCRATCH/claude-config claude plugin marketplace add $WORKTREE
CLAUDE_CONFIG_DIR=$SCRATCH/claude-config claude plugin install pr-evidence@pr-evidence
CLAUDE_CONFIG_DIR=$SCRATCH/claude-config claude plugin list
```

Expected: `pr-evidence@pr-evidence` is installed and enabled.

- [ ] **Step 5: Gemini CLI waits for the push**

Overriding `HOME` is refused in a worktree-isolated session; Gemini's own `GEMINI_CLI_HOME` works instead. A local-path install stops at Gemini's folder-trust prompt, so Gemini installs from the pushed branch in Task 5.

### Task 4: README

**Files:**
- Modify: `README.md`, the Install table and the Development section

**Interfaces:**
- Consumes: the Cursor install command from Task 3, Step 2.

- [ ] **Step 1: Add the Cursor row after the Copilot CLI row**

```markdown
| ![Cursor][badge-cursor] | `agent plugin marketplace add https://github.com/ArefMozafari/pr-evidence`<br>then `/plugins` in `agent` and install `pr-evidence` |
```

- [ ] **Step 2: Remove `![Cursor][badge-cursor] ` from the `npx skills add` row**

The row becomes:

```markdown
| ![Codex][badge-codex] ![Windsurf][badge-windsurf] ![Cline][badge-cline] and more | `npx skills add ArefMozafari/pr-evidence` |
```

- [ ] **Step 3: Add the app route under the table**

```markdown
In the Cursor app, open **Customize**, choose **From GitHub Repository**, and paste the repository URL.
```

The Cursor CLI cannot check this route, so the owner checks it in the app after the merge (Task 6, Step 4). That check failed, so the line was removed again; see #9.

- [ ] **Step 4: Make the Development section match the workflow**

Change `To run the same checks locally from the repository root:` to `To run the skill and Claude Code checks locally from the repository root:`, and add after the code block:

```markdown
The workflow also checks the Gemini manifest's fields and validates the Cursor manifests against
the JSON schemas Cursor publishes.
```

- [ ] **Step 5: Render the README and look at it**

```bash
bash $SCRATCH/render-readme.sh
```

Expected: the screenshots are exactly 1280 px and 390 px wide; the Cursor row shows its badge and both commands.

### Task 5: Commit, push, and the checks that need GitHub

- [ ] **Step 1: Stop and report.** Wait for "commit and push".

- [ ] **Step 2: Commit in four commits and push**

```bash
git add .cursor-plugin/plugin.json .cursor-plugin/marketplace.json
git commit -m "feat: add a Cursor plugin manifest and marketplace"
git add .github/workflows/validate.yml
git commit -m "ci: validate the Cursor manifests against Cursor's schemas"
git add README.md
git commit -m "docs: add the Cursor plugin install to the README"
git add docs/plans/2026-09-26-pr-evidence-repo-design.md docs/plans/2026-09-28-cursor-plugin-plan.md
git commit -m "docs: record the Cursor plugin design and plan"
git push -u origin feature/cursor-plugin-manifest
```

Expected: the Validate run on the branch passes, including the new Cursor step.

- [ ] **Step 3: The Cursor CLI indexes the branch with both manifests present**

```bash
agent plugin marketplace add https://github.com/ArefMozafari/pr-evidence --git-ref feature/cursor-plugin-manifest
```

Expected: `✓ Added marketplace pr-evidence (1 plugin)`. Then remove the marketplace, so the owner's Cursor setup is as before.

- [ ] **Step 4: Gemini CLI and Copilot CLI still install from the branch**

```bash
GEMINI_CLI_HOME=$SCRATCH/home-gemini npx -y @google/gemini-cli@0.61.0 extensions install https://github.com/ArefMozafari/pr-evidence --ref feature/cursor-plugin-manifest --consent
GEMINI_CLI_HOME=$SCRATCH/home-gemini npx -y @google/gemini-cli@0.61.0 extensions list
```

Expected: `pr-evidence` is installed and enabled. For Copilot, read `npx -y @github/copilot@1.0.88 plugin marketplace add --help` for a branch option and a home-folder setting that is not `HOME`. With both, install `pr-evidence@pr-evidence` from the branch. Otherwise run the check against `main` after the merge and say so in the pull request.

- [ ] **Step 5: Record the results in the design note**

Add to `## Verification` in `docs/plans/2026-09-26-pr-evidence-repo-design.md` one bullet for the Cursor schema check at `cursor/plugins` commit `ecc249f`, and one for the installs, in the words of what was observed. Commit as `docs: record the Cursor plugin install checks` after approval.

### Task 6: Evidence and pull request

- [ ] **Step 1: Capture the after screenshots of the Install table on the branch**, logged out in a fresh context: light at 1280 px and dark at 390 px, the same as the before screenshots taken on `main`.

- [ ] **Step 2: Stop and report.** Wait for "push evidence and open the PR".

- [ ] **Step 3: Push the four screenshots to `pr-assets` under a fresh `<pr>/1/` folder**, and open the pull request with the six-section body, `.cursor-plugin/marketplace.json` as the snippet, the install results, and `Closes #1`. The body says the app route is checked after the merge.

- [ ] **Step 4: After the merge, the owner imports the repository in the Cursor app again**

Expected: `agent plugin marketplace update arefmozafari-pr-evidence` reports 1 plugin indexed, and the app lists `pr-evidence` with an Install button. If it reports 0, file a `[BUG]` issue and correct the README.
