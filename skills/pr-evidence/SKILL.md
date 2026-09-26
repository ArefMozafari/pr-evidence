---
name: pr-evidence
description: How a PR shows its change instead of describing it - before/after screenshots for UI work captured in a clean environment (emulator, simulator, or fresh browser profile) with a throwaway account, hosting images on a non-merging pr-assets branch with a fresh path per capture, the one-snippet rule for non-UI changes, and the six-section PR body. Use when opening or drafting a PR or MR, writing a PR description, attaching screenshots, or at the start of any UI-facing unit so the "before" baseline is captured on the base branch first.
---

# PR evidence — show the change, don't describe it

A reviewer should grasp what moved without reading the diff.

## What to show

- **UI/UX change → before *and* after screenshots**, same screen, same device or viewport, same
  state.
- **Everything else → the one short snippet that *is* the change** — the few lines that
  carry it, not a tour of every file.

## Capture the "before" first

Capture the baseline **while still on the base branch, at the start of the unit**.
Reconstructing it afterwards costs a second build, install, and re-drive of the app. If a unit
turns out to be UI-facing after work has started, capture the baseline before going further.

## Where to capture

**In a clean environment with a throwaway account, never on your personal device or browser
profile.** For mobile work that means an emulator or simulator; for web work, a fresh browser
profile. A personal device holds real sessions, and scrubbing it means destroying them. A clean
environment starts empty, is disposable, and nothing private can leak into a public repo. Keep
hands-on testing on a real device where it matters — that is about whether it works; this is
about what a reviewer sees.

Screenshots are published artifacts. On a **public** repo they carry whatever is on screen —
account names, server hostnames, room IDs, message content. Check the frame before attaching,
and ask if real data can't be avoided.

## Hosting images on GitHub

GitHub has no API for PR image attachments — the drag-and-drop `user-attachments` flow is
browser-session only, so `gh` cannot do it. Images need a public URL first:

1. Push them to a **non-merging `pr-assets` branch**, so binaries never enter the main history.
2. Reference `https://github.com/<owner>/<repo>/blob/pr-assets/<pr>/<rev>/<name>.png?raw=true`.
   That form renders for anyone with access to the repo, private repos included;
   `raw.githubusercontent.com` links only render on public repos.
3. Lay them out as a two-column before/after table.

**Never overwrite an image path.** GitHub proxies external images through its camo cache, so
replacing a file at a URL that has already been rendered can keep serving the stale picture.
Give every capture a fresh path (`<pr>/<rev>/<name>.png`) instead of re-uploading over an old
one.

## PR body

Use the repo's own template if it has one. Otherwise these six sections, in this order:

1. **Task summary**
2. **Scope of work** — a checklist
3. **Testing instructions**
4. **Screenshots** — UI changes; the before/after table goes here
5. **Breaking changes** — yes/no, and what
6. **Documentation / links**

Reference issues as "tracked in #NN" unless the PR should close them — `Closes/Fixes/Resolves
#NN` auto-closes on merge.
