# README hero comparison implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show, directly under the README header, one pull request's description without and with the skill, as issue #2 asks.

**Architecture:** Two Markdown sources in `docs/hero/` are rendered by GitHub's Markdown API in `gfm` mode, wrapped in a bordered frame styled with GitHub's Markdown CSS, and captured with the Playwright CLI into four PNGs in `docs/assets/`. The README shows them in the same `<table>` of `<picture>` elements the Example section uses. The design and its rejected alternatives are in `docs/plans/2026-09-26-readme-visual-design.md`, section "Hero comparison".

**Tech Stack:** Markdown and HTML, `gh api markdown`, `github-markdown-css@5.9.0` from jsDelivr, `playwright@1.63.0` driving the installed Google Chrome.

## Global Constraints

- Every commit, push, and pull request waits for the user's explicit approval; the steps below marked **Gate** stop there.
- No reference to Claude, Anthropic, or AI in commits, pull request text, code, comments, or docs.
- Commit subjects follow the repository's history: `docs: <imperative, lower case>`, no body needed.
- `gh` must be signed in as `ArefMozafari` (`gh auth status`); the identity guard blocks `gh api markdown` otherwise, because it is a POST.
- Frames are 480 px wide and captured at 2×, so every hero PNG is 960 px wide; all four share one height.
- Pinned versions: `github-markdown-css@5.9.0`, `playwright@1.63.0`, Chrome through `--channel chrome`.
- Every capture runs in a fresh browser context: logged out, no profile, nothing private on screen.
- Frame colours come from `github-markdown-css@5.9.0`: light background `#ffffff` and border `#d1d9e0`, dark background `#0d1117` and border `#3d444d`.
- Pull request evidence goes to `pr-assets` under a path never used before; the next number is 5, so `5/1/`.
- Identifiers in any script are full words, no abbreviations; comments sit above the line they describe.
- `$SCRATCH` below is any directory outside the repository.

## File structure

| Path | Change | Responsibility |
| --- | --- | --- |
| `docs/hero/without.md` | Create | Pull request #3 written the way agents write it without the skill |
| `docs/hero/with.md` | Create | Verbatim excerpt of pull request #3's real description |
| `docs/assets/hero-{without,with}-{light,dark}.png` | Create | The four rendered frames |
| `README.md` | Modify, after line 23 | Hero table and caption under the header |
| `docs/plans/2026-09-26-readme-visual-design.md` | Modify | Regeneration steps and hero verification |
| `docs/plans/2026-09-28-readme-hero-comparison-plan.md` | Create | This plan |

There is no application code, so there are no unit tests. Each task's check is looking at the rendered output, and the repository's validation commands must keep passing.

---

### Task 1: Capture the "before" evidence on `main`

The skill requires the baseline before any README edit.

**Files:**
- Create: `$SCRATCH/capture/capture.mjs` (not committed)
- Output: `$SCRATCH/evidence/readme-before-light.png`, `$SCRATCH/evidence/readme-before-dark.png`

**Interfaces:**
- Produces: `node capture.mjs <url> <viewport width> <clip height> <light|dark> <output path>`, reused by Task 5.

- [ ] **Step 1: Install Playwright in a scratch project**

```bash
mkdir -p "$SCRATCH/capture" "$SCRATCH/evidence"
cd "$SCRATCH/capture" && npm init -y && npm install playwright@1.63.0
```

Expected: `added N packages`, no browser download, because Chrome is used through its channel.

- [ ] **Step 2: Write the capture script**

`$SCRATCH/capture/capture.mjs`:

```js
// Captures the top of a GitHub repository's rendered README in a fresh, logged-out browser context.
import { chromium } from 'playwright';

// Keeps the whole clip inside the viewport, so lazily loaded images are loaded before capture.
const viewportHeight = 2400;

const [url, viewportWidth, clipHeight, colorScheme, outputPath] = process.argv.slice(2);

const browser = await chromium.launch({ channel: 'chrome' });
const context = await browser.newContext({
  viewport: { width: Number(viewportWidth), height: viewportHeight },
  colorScheme,
});
const page = await context.newPage();
// Waits for load, not network idle, because GitHub keeps background requests open.
await page.goto(url, { waitUntil: 'load' });
const readmeBounds = await page.locator('article.markdown-body').first().boundingBox();
const clipBottom = readmeBounds.y + Number(clipHeight);
// Waits until every README image and Mermaid diagram that starts inside the clip has finished rendering.
await page.waitForFunction((bottom) => {
  const startsInsideClip = (element) => element.getBoundingClientRect().top < bottom;
  const imagesLoaded = [...document.querySelectorAll('article.markdown-body img')]
    .filter(startsInsideClip)
    .every((image) => image.complete && image.naturalWidth > 0);
  const diagramsRendered = [...document.querySelectorAll('article.markdown-body .js-render-target')]
    .filter(startsInsideClip)
    .every((diagram) => diagram.classList.contains('is-render-ready'));
  return imagesLoaded && diagramsRendered;
}, clipBottom);
await page.screenshot({
  path: outputPath,
  clip: {
    x: readmeBounds.x,
    y: readmeBounds.y,
    width: readmeBounds.width,
    height: Math.min(readmeBounds.height, Number(clipHeight)),
  },
});
await browser.close();
```

- [ ] **Step 3: Capture light at 1280 px and dark at 390 px**

```bash
cd "$SCRATCH/capture"
node capture.mjs https://github.com/ArefMozafari/pr-evidence/tree/main 1280 1000 light "$SCRATCH/evidence/readme-before-light.png"
node capture.mjs https://github.com/ArefMozafari/pr-evidence/tree/main 390 1000 dark "$SCRATCH/evidence/readme-before-dark.png"
```

Expected: two PNGs, no error.

- [ ] **Step 4: Inspect both images**

Open each PNG. Pass: it starts at the README's `pr-evidence` heading and shows the header badges and the start of "What it teaches your agent"; it has no avatar, no signed-in menu, and nothing private. If the clip height cuts the header, raise it and recapture both; Task 5 must use the same value. `main` is at `126a503`; if `main` moves before Task 5, recapture from `tree/126a503` instead.

---

### Task 2: Frame sources and hero images

**Files:**
- Create: `docs/hero/without.md`, `docs/hero/with.md`
- Create: `$SCRATCH/render-hero.sh` (its text goes into the design note in Task 4)
- Create: `docs/assets/hero-without-light.png`, `docs/assets/hero-without-dark.png`, `docs/assets/hero-with-light.png`, `docs/assets/hero-with-dark.png`

**Interfaces:**
- Produces: the four PNG paths above, used by Task 3, and the final `frame_height` value, recorded in Task 4.

- [ ] **Step 1: Write `docs/hero/without.md`**

```markdown
## Summary

- Redesign `README.md` with a centered header, status badges, and agent badges
- Convert the rules list to a Rule/Why table
- Add a Mermaid workflow diagram
- Collapse the manual install and development sections
- Add `docs/plans/2026-09-26-readme-visual-design.md`

## Test plan

- [x] Checked that the README renders on GitHub
- [x] Verified that the badges load
```

- [ ] **Step 2: Write `docs/hero/with.md`**

Copied verbatim from pull request #3's description (`gh pr view 3 --json body -q .body`), Task summary and Screenshots sections only:

```markdown
## Task summary

Redesigns the README so it scans at a glance instead of reading as a wall of text: a centered header with badges, a Rule/Why table, a workflow diagram, and collapsed sections for the details most readers skip. The screenshots below were captured by following this repository's own skill.

## Screenshots

Captured logged out in a fresh browser profile at 1280 px, on `main` (before) and on this branch (after).

| Before | After |
| --- | --- |
| ![Before, light](https://github.com/ArefMozafari/pr-evidence/blob/pr-assets/3/1/readme-before-light.png?raw=true) | ![After, light](https://github.com/ArefMozafari/pr-evidence/blob/pr-assets/3/1/readme-after-light.png?raw=true) |
| ![Before, dark](https://github.com/ArefMozafari/pr-evidence/blob/pr-assets/3/1/readme-before-dark.png?raw=true) | ![After, dark](https://github.com/ArefMozafari/pr-evidence/blob/pr-assets/3/1/readme-after-dark.png?raw=true) |
```

Check it is verbatim: `diff` the two sections against the live description; the only allowed difference is the omitted sections.

- [ ] **Step 3: Write the render script**

`$SCRATCH/render-hero.sh`:

```bash
#!/usr/bin/env bash
# Renders the README hero frames into docs/assets/. Run it from the repository root.
set -euo pipefail

# Shows the top of pull request #3's screenshots, badge rows included, above the fade.
frame_height=600
work_directory=$(mktemp -d)

for frame in without with; do
  gh api markdown -f mode=gfm -f context=ArefMozafari/pr-evidence \
    -F text=@"docs/hero/$frame.md" > "$work_directory/$frame.html"
  for theme in light dark; do
    if [ "$theme" = light ]; then
      border_color='#d1d9e0'
      background_color='#ffffff'
    else
      border_color='#3d444d'
      background_color='#0d1117'
    fi
    page="$work_directory/$frame-$theme-page.html"
    cat > "$page" <<EOF
<!doctype html>
<meta charset="utf-8">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/github-markdown-css@5.9.0/github-markdown-$theme.css">
<style>
  :root {
    --frame-border: $border_color;
    --frame-background: $background_color;
    --frame-padding: 16px;
    --frame-radius: 6px;
    --fade-height: 96px;
  }
  .frame {
    box-sizing: border-box;
    position: relative;
    height: 100vh;
    overflow: hidden;
    padding: var(--frame-padding);
    border: 1px solid var(--frame-border);
    border-radius: var(--frame-radius);
  }
  .frame > :first-child {
    margin-top: 0;
  }
  .frame::after {
    content: "";
    position: absolute;
    right: 0;
    bottom: 0;
    left: 0;
    height: var(--fade-height);
    background: linear-gradient(transparent, var(--frame-background));
  }
</style>
<body class="markdown-body">
<article class="frame">
$(cat "$work_directory/$frame.html")
</article>
</body>
EOF
    npx -y playwright@1.63.0 screenshot --channel chrome --device "Desktop Chrome HiDPI" \
      --viewport-size "480, $frame_height" --color-scheme "$theme" \
      "file://$page" "docs/assets/hero-$frame-$theme.png"
  done
done
```

- [ ] **Step 4: Run it**

```bash
bash "$SCRATCH/render-hero.sh"
sips -g pixelWidth -g pixelHeight docs/assets/hero-*.png
```

Expected: four PNGs, each `pixelWidth: 960` and `pixelHeight: 1200` (2 × 600).

- [ ] **Step 5: Inspect and tune the height**

Open all four. Pass:
- Left frame: both headings, five bullets, two ticked checkboxes, nothing cut off.
- Right frame: Task summary, the Screenshots heading and caption, the Before/After header, and the top of both light screenshots, with the after image's badge rows visible above the fade.
- Dark frames: dark background and border, readable text, no white page edge.

If the badge rows sit under the fade, raise `frame_height` by 40 and rerun Step 4; if the right frame shows much more than the top of the screenshots, lower it by 40. Record the final value in the script. (640 showed the after screenshot down to its diagram; 600 keeps the badge rows about 60 px above the fade and is the value used.)

- [ ] **Step 6: Gate: commit the sources, then the images**

```bash
git add docs/hero/without.md docs/hero/with.md
git commit -m "docs: add the README hero frame sources"
git add docs/assets/hero-without-light.png docs/assets/hero-without-dark.png docs/assets/hero-with-light.png docs/assets/hero-with-dark.png
git commit -m "docs: add the README hero comparison images"
```

---

### Task 3: Hero table in the README

**Files:**
- Modify: `README.md`, inserting after line 23 (`</div>` closing the header) and before `## What it teaches your agent`

**Interfaces:**
- Consumes: the four PNG paths from Task 2.

- [ ] **Step 1: Insert the hero**

Between `</div>` and `## What it teaches your agent`, with one blank line on each side:

`width="50%"` on the header cells keeps the two columns equal. Without it, the columns size from their header text, the frames scale to different heights, and the cells' vertical centring offsets them. GitHub keeps the attribute; `gh api markdown` returns it unchanged.

```html
<table>
  <tr>
    <th width="50%">Without the skill</th>
    <th width="50%">With the skill</th>
  </tr>
  <tr>
    <td>
      <picture>
        <source media="(prefers-color-scheme: dark)" srcset="docs/assets/hero-without-dark.png">
        <img src="docs/assets/hero-without-light.png" alt="A pull request description without the skill: a Summary listing the changed files and a ticked Test plan, with no images">
      </picture>
    </td>
    <td>
      <picture>
        <source media="(prefers-color-scheme: dark)" srcset="docs/assets/hero-with-dark.png">
        <img src="docs/assets/hero-with-light.png" alt="The same pull request's real description with the skill: a Task summary, then a Screenshots table showing the README before and after side by side">
      </picture>
    </td>
  </tr>
</table>

The same change, [pull request #3](https://github.com/ArefMozafari/pr-evidence/pull/3), written up both ways.
The right side is an excerpt of its real description, rendered with GitHub's Markdown styles.
```

- [ ] **Step 2: Render the README locally in light at 1280 px and dark at 390 px**

From the worktree root. `mode=markdown` is how GitHub renders README files. The `<base>` element resolves `docs/assets/` against the worktree, and the 830 px column approximates the README column on github.com at 1280 px; Task 5 checks the real one.

```bash
gh api markdown -f mode=markdown -F text=@README.md > "$SCRATCH/readme-body.html"
worktree=$(pwd)
for theme in light dark; do
  {
    printf '<!doctype html>\n<meta charset="utf-8">\n<base href="file://%s/">\n' "$worktree"
    printf '<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/github-markdown-css@5.9.0/github-markdown-%s.css">\n' "$theme"
    printf '<body class="markdown-body" style="max-width: 830px; margin: 0 auto; padding: 32px;">\n'
    cat "$SCRATCH/readme-body.html"
    printf '</body>\n'
  } > "$SCRATCH/readme-$theme.html"
done
```

```bash
npx -y playwright@1.63.0 screenshot --channel chrome --viewport-size "1280, 900" --color-scheme light --full-page "file://$SCRATCH/readme-light.html" "$SCRATCH/readme-local-light.png"
npx -y playwright@1.63.0 screenshot --channel chrome --viewport-size "390, 844" --color-scheme dark --full-page "file://$SCRATCH/readme-dark.html" "$SCRATCH/readme-local-dark.png"
sips -g pixelWidth "$SCRATCH/readme-local-light.png" "$SCRATCH/readme-local-dark.png"
```

Expected: widths `1280` and `390`, meaning no horizontal page scroll. Open both: the hero sits under the badges, the two frames line up, and the caption follows the table. Two differences from github.com are expected locally: the Mermaid block shows as code, and the dark render still shows the light images. The API wraps each `<img>` in a link, so it is no longer a direct child of `<picture>`, and only github.com's `<themed-picture>` script restores the theme switch. Task 5 checks the switch on the live page.

- [ ] **Step 3: Gate: commit**

```bash
git add README.md
git commit -m "docs: show a with/without comparison under the README header"
```

---

### Task 4: Design note and validation

**Files:**
- Modify: `docs/plans/2026-09-26-readme-visual-design.md`
- Add: this plan

- [ ] **Step 1: Add the regeneration steps**

Under the "Hero comparison" bullets, add `### Regenerating the hero`, one sentence saying to run it from the repository root with `gh` signed in, and the final `render-hero.sh` from Task 2 in a `bash` block, with the tuned `frame_height`.

- [ ] **Step 2: Add the hero verification**

Append to `## Verification` what Tasks 2 and 3 observed: the PNG sizes, the local render widths, and the theme switch. Write only what was run.

- [ ] **Step 3: Run the repository's validation**

```bash
uvx --from skills-ref==0.1.1 agentskills validate skills/pr-evidence
claude plugin validate --strict .claude-plugin/marketplace.json
claude plugin validate --strict .claude-plugin/plugin.json
```

Expected: `Valid skill: skills/pr-evidence` and `✔ Validation passed` twice.

- [ ] **Step 4: Gate: commit**

```bash
git add docs/plans/2026-09-26-readme-visual-design.md docs/plans/2026-09-28-readme-hero-comparison-plan.md
git commit -m "docs: record the README hero design and plan"
```

---

### Task 5: Live check, evidence, pull request

- [ ] **Step 1: Gate: push the branch**

```bash
git push -u origin improve/readme-hero-comparison
```

- [ ] **Step 2: Capture the "after" evidence**

The same script, clip height, widths, and themes as Task 1:

```bash
cd "$SCRATCH/capture"
node capture.mjs https://github.com/ArefMozafari/pr-evidence/tree/improve/readme-hero-comparison 1280 1000 light "$SCRATCH/evidence/readme-after-light.png"
node capture.mjs https://github.com/ArefMozafari/pr-evidence/tree/improve/readme-hero-comparison 390 1000 dark "$SCRATCH/evidence/readme-after-dark.png"
```

Open both. Pass: the hero renders under the badges on real GitHub, in the right theme, with no horizontal scroll at 390 px.

Then confirm the theme switch file by file with `$SCRATCH/capture/themed.mjs`:

```js
// Prints which image file each README picture shows on github.com in a given color scheme.
import { chromium } from 'playwright';

const [url, colorScheme] = process.argv.slice(2);

const browser = await chromium.launch({ channel: 'chrome' });
const context = await browser.newContext({ viewport: { width: 1280, height: 4000 }, colorScheme });
const page = await context.newPage();
await page.goto(url, { waitUntil: 'load' });
// Waits until every README picture has loaded its image.
await page.waitForFunction(() =>
  [...document.querySelectorAll('article.markdown-body picture img')].every(
    (image) => image.complete && image.naturalWidth > 0,
  ),
);
const shownFiles = await page.evaluate(() =>
  [...document.querySelectorAll('article.markdown-body picture img')].map((image) =>
    decodeURIComponent(image.currentSrc.split('/').pop().split('?')[0]),
  ),
);
console.log(colorScheme, shownFiles);
await browser.close();
```

```bash
node themed.mjs https://github.com/ArefMozafari/pr-evidence/tree/improve/readme-hero-comparison light
node themed.mjs https://github.com/ArefMozafari/pr-evidence/tree/improve/readme-hero-comparison dark
```

Expected: `light [ 'hero-without-light.png', 'hero-with-light.png', 'example-before-light.png', 'example-after-light.png' ]`, and the same with `dark` in each name. On `main` before this change it printed only the two Example files, switching the same way.

- [ ] **Step 3: Confirm the pull request number**

```bash
gh api "repos/ArefMozafari/pr-evidence/issues?state=all&per_page=1" -q '.[0].number'
```

Expected: `4`, so the pull request will be 5. If it is higher, use that number plus one in every `5/1/` path below.

- [ ] **Step 4: Gate: push the evidence to `pr-assets` without checking it out**

One command per line; paste each SHA into the next command.

```bash
export GIT_INDEX_FILE="$SCRATCH/pr-assets.index"
git read-tree origin/pr-assets
git hash-object -w "$SCRATCH/evidence/readme-before-light.png"
git update-index --add --cacheinfo 100644,<sha>,5/1/readme-before-light.png
```

Repeat `hash-object` and `update-index` for `readme-before-dark.png`, `readme-after-light.png`, and `readme-after-dark.png`, then:

```bash
git write-tree
git commit-tree <tree sha> -p origin/pr-assets -m "Add README hero screenshots for pull request 5"
git push origin <commit sha>:refs/heads/pr-assets
unset GIT_INDEX_FILE
```

Expected: `git ls-tree -r --name-only origin/pr-assets` (after `git fetch`) lists `3/1/`, `4/1/`, and the four new `5/1/` files.

- [ ] **Step 5: Gate: open the pull request**

Title: `docs: show a with/without comparison under the README header`. The body uses the skill's six sections:
1. Task summary
2. Scope of work checklist
3. Testing instructions: open the branch README; switch light and dark; narrow to phone width
4. Screenshots: a Before/After table of the four `pr-assets/5/1/` images, captured logged out in a fresh browser context at 1280 px (light) and 390 px (dark)
5. Breaking changes: no
6. Documentation / links: the design note and this plan

It ends with `Closes #2`, since every box on the issue is covered.

- [ ] **Step 6: After opening**

Check CI's Validate run through the pull request status tools, then remind the user that this is a clean point to run `/compact`.
