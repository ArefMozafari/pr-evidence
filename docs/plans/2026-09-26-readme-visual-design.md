# A scannable, visual README

Date: 2026-09-26

## Goal

Make the README readable at a glance instead of as a wall of text, adding image files only where
a rendered picture is the point.

## Decision

- **Centered header** with a one-line tagline, a row of status badges (CI, release, license,
  Agent Skills), and a row of "works with" badges that link to the install table.
- **Badges are reference-style links defined once** at the bottom of the README. The header and
  the install table reuse the same definitions, so each badge has one look and one URL.
- **Every agent badge shares one style**: the `24292F` background with a white logo. Brand colours
  were rejected because several of these brands' official colour is black, so the row would mix
  near-black and bright badges.
- **The Codex badge embeds the simple-icons OpenAI SVG as a base64 logo**, because the icon set
  shields.io serves has no `openai` entry and would silently render the badge without a logo.
- **"What it teaches" is a Rule/Why table** rather than bullets, so each rule and its reason read
  on one row.
- **"How it works" is a Mermaid diagram**, which GitHub renders natively and themes for light
  and dark mode. It flows top-down: a left-to-right layout scaled its labels down until they
  were unreadable at phone width.
- **Manual install and Development sit in collapsed `<details>` blocks**, since most readers
  only need the one-command install.
- **The Example is rendered, not quoted.** A Markdown snippet of a before/after table showed the
  syntax but not the result, which is the one thing this skill is about. The Example is now a
  real before/after table built from this README's own redesign (pull request #3): crops of the
  top of the README at the commit before and the commit after, in light and dark variants that a
  `<picture>` element switches with the reader's theme. The snippet stays, collapsed, for readers
  who want the syntax.
- **README images live in `docs/assets/`, not on `pr-assets`.** The `pr-assets` branch holds
  evidence for a pull request's review; the README depends on these images permanently, so they
  belong in the main history with the page that shows them.

## Hero comparison

- **A hero under the header shows one pull request without and with the skill.** The README
  explained the rules but never showed their payoff at the level of a whole pull request. The
  hero reuses the Example's shape, a two-column `<table>` of `<picture>` elements headed
  "Without the skill" and "With the skill", so both comparisons on the page look alike. The
  Example stays as it is: it shows the same screenshots at a readable size.
- **The hero's header cells carry `width="50%"`**, which GitHub keeps. Without it the columns
  size from their header text, the two frames scale to different heights, and the cells'
  vertical centring offsets them.
- **Both frames describe one real change, pull request #3.** The right frame is a verbatim
  excerpt of its description, the Task summary and Screenshots sections. The left frame is the
  same change written the way agents write pull requests without the skill: a Summary listing
  the touched files and a ticked Test plan, with no images. A careful prose description was
  rejected as less familiar to readers; the real description minus its Screenshots section was
  rejected because it keeps the skill's six-section structure and undersells the difference.
- **The frames are rendered, not screenshots of pull request pages.** Each frame's Markdown
  lives in `docs/hero/` and goes through GitHub's Markdown API in `gfm` mode with this
  repository as context, so `#3` renders as a link, then is styled with GitHub's Markdown CSS
  inside a bordered box. Both frames get the same width and theme, no throwaway pull requests
  are opened, and no avatar or username appears in the image. A one-line caption under the
  table says the right frame is an excerpt of pull request #3's real description.
- **Both frames share one fixed size**: 480 × 600 px, captured at 2×. Pull request #3's
  screenshots are full-page captures, 1265 px wide and 2362 px (before) and 2926 px (after)
  tall, so the right frame is cut at 600 px with a fade marking it as an excerpt. 600 px keeps
  the after screenshot's badge rows above the fade; 640 px showed it down to its diagram. The
  cut falls inside the first screenshot row, so both themes show the light row, which is also
  the row the real description shows first.
- **The hero images live in `docs/assets/`** as `hero-without-light.png`,
  `hero-without-dark.png`, `hero-with-light.png`, and `hero-with-dark.png`, for the same reason
  as the Example's images.
- **No render script is committed.** Four images do not justify code to maintain; the steps to
  regenerate them are recorded in this note instead.

### Regenerating the hero

Run this from the repository root with `gh` signed in and Google Chrome installed. It
overwrites the four `docs/assets/hero-*.png` files.

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

The colours are the background and border colours of `github-markdown-css@5.9.0`'s light and
dark files; the stylesheet defines no custom properties to reuse.

## Verification

- Every badge and link URL returns HTTP 200, and every logo badge contains its embedded image.
- The README was rendered through GitHub's Markdown API in `markdown` mode (the mode GitHub
  uses for README files; `gfm` mode turns line breaks into `<br>` and stacks the badges), then
  checked in light mode at 1280 px and in dark mode at 390 px, with no horizontal page scroll.
- The hero's `docs/hero/with.md` matches the Task summary and Screenshots sections of pull
  request #3's live description character for character. The four hero images are 960 × 1200
  px. With the hero in place, the README rendered as above is exactly 1280 px and 390 px wide,
  and the two frames line up.
- A local render cannot show the theme switch: the Markdown API wraps each `<img>` in a link,
  so it is no longer a direct child of `<picture>`, and only github.com's `<themed-picture>`
  script restores the switch. On github.com, in a fresh browser context, the Example's pictures
  load their `-light.png` files in light mode and their `-dark.png` files in dark mode.
