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

## Deferred

A hero image comparing a text-only PR with a before/after PR is left for a follow-up, tracked
as its own issue.

## Verification

- Every badge and link URL returns HTTP 200, and every logo badge contains its embedded image.
- The README was rendered through GitHub's Markdown API in `markdown` mode (the mode GitHub
  uses for README files; `gfm` mode turns line breaks into `<br>` and stacks the badges), then
  checked in light mode at 1280 px and in dark mode at 390 px, with no horizontal page scroll.
