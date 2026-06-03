# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The **Feeding Report — TV Display**: a single, self-contained HTML file (`Feedingreport_display.html`, inline CSS + JS, no framework/build) shown full-screen on a TV/monitor in the daycare. It is a **read-only dashboard** — it shows, per pen, which dogs are being fed this session, their feeding amount (All / ¾ / ½ / ¼ / None), and medicine/supplement flags. Staff never interact with it; it just mirrors what the feeding tablet is doing, live.

It has **no backend, no database, and no write path of its own.** Everything it shows comes from polling another project's web app. **This folder cannot work in isolation** — see the dependency section next.

## ⚠️ Hard dependency: the Feeding Manager project (the "other folder")

This display is a **thin, read-only client of the Feeding Manager system**, which lives in the sibling folder:

```
…\CODING\Telegram_feeding manager\     ← the OTHER folder (owns everything this display reads)
…\CODING\Feeding report display\        ← THIS folder (display only)
```

Read `..\Telegram_feeding manager\CLAUDE.md` first — it documents the backend, the Session sheet, and the tablet. This display depends on three things that project owns:

1. **The Google Apps Script (GAS) web app.** The constant `APPS_SCRIPT_URL` at the top of the script points at the **same `/exec` deployment the feeding tablet uses** (Apps Script project "Feeding manager", container-bound to the feeding Google Sheet). This display only ever issues **GET** requests to it — it never POSTs.
   - If that GAS deployment's URL ever changes (e.g. someone runs a fresh `clasp deploy` instead of `clasp redeploy`, which mints a **new** `/exec` URL), this display goes blank until `APPS_SCRIPT_URL` here is updated to match. The correct deploy in the other project is always `clasp redeploy <deploymentId>` precisely so this URL stays stable.

2. **The shared Session sheet + the `getSession` contract.** The Feeding Manager's backend writes a per-dog row to the **Session** tab as staff edit on the tablet; this display reads it back. The two read endpoints it consumes (both must keep their shape):
   - **`?action=getSession`** → `{ success, mealType, version, count, dogs: [ … ] }`, where each dog is
     `{ id, inputName, matchedName, possibleMatches, status, prescription, prescriptionComment, supplements, supplementTypes, penId, position, lastUpdated }`.
     This display reads: `matchedName`/`inputName` (name), `status` (feeding amount), `prescription`/`supplements` (the `+P`/`+S` indicators), `penId` (which pen), and **`position`** (within-pen order — see below).
   - **`?action=getSessionVersion`** → `{ success, version, count }` (lightweight change-detection for polling).
   - **Pen IDs must match exactly:** `top-1`…`top-5`, `bottom-1`…`bottom-5` (this display pre-creates those 10 pens; a dog with any other `penId` is dropped).
   - **`status` values must match:** `all | three-quarter | half | quarter | none` (mapped to labels via `STATUS_LABELS`; CSS classes `status-<value>` set the card colour).

3. **Within-pen feeding order via the Session `Position` column (added 2026-06-03).** The tablet lets staff drag dogs up/down within a pen; that order is persisted as the Session `Position` column and returned by `getSession` as `dog.position`. This display **sorts each pen by `position`** (see `applyData`) so the TV shows the **same feeding order** as the tablet. Legacy/0 positions keep server-row order (stable sort), so it degrades gracefully if `position` is ever absent.

### Cross-folder contract — change both sides together

Anything in the Feeding Manager project that changes the above will silently break this display. Specifically, if that project:
- changes the GAS `/exec` deployment URL → update `APPS_SCRIPT_URL` here;
- renames/removes `getSession` or `getSessionVersion`, or drops a field this display reads → update the consumer here;
- changes the pen-ID strings or the `status` value set → update `pens{}` / `STATUS_LABELS` here;
- removes or renames the Session `Position` column / `dog.position` → this display falls back to unsorted within-pen order (the sort becomes a no-op), so re-derive ordering if the source changes.

This display is purely downstream: it adds no data and writes nothing back. When in doubt, the Feeding Manager project is the source of truth.

## How it works (architecture)

- **Bootstrap** (`DOMContentLoaded`): `renderPensStructure()` builds the 10 empty pen containers (ids `dogs-<penId>`), then `loadData()` does the first fetch, then the adaptive auto-refresh starts.
- **`loadData()`** → GET `?action=getSession` → `applyData(result)`.
- **`applyData(result)`**: sets `currentMealType`; tracks `lastKnownVersion`/`lastKnownCount`; rebuilds `dogs` + the `pens{}` map (objects keyed by pen ID, each an array of dog objects); **sorts each pen by `dog.position`** (stable; ties keep server order) so the order matches the tablet; then `renderAllPens()`, `updateStats()`, `calculateScale()`.
- **Rendering** (`renderPen`): a `.dog-card status-<status>` per dog with the name, `+P`/`+S` indicators, and the feeding-amount label.
- **Dynamic TV scaling** (`calculateScale`): picks a `scale-xl … scale-xs` class on `#tvContainer` from the **max dogs in any one pen**, so the layout fits the screen without scrolling regardless of how full the busiest pen is.
- **Adaptive auto-refresh** (quota-friendly): normal mode polls the lightweight `getSessionVersion` every **10s**; when `version` or `count` changes it does a full `getSession` load and switches to **fast mode** (full reload every **3s**) for **30s**, then reverts to normal. A failed version check falls back to a full `loadData()` to stay resilient. Net effect: the TV reflects a tablet change within a few seconds, but sits at a cheap 10s heartbeat when nothing is happening.
- **No `localStorage`, no mutation queue, no offline model** — unlike the tablet, this is a stateless viewer. A failed fetch just retries on the next tick; there is nothing to lose.

## Deployment

This on-disk file is the **editable source but is NOT git-tracked** (it lives in OneDrive). It ships to its **own GitHub Pages repo, `Fairytails123/frmdisplay`**, served as **`index.html`** (note: the repo file is `index.html`, the working file here is `Feedingreport_display.html` — same content, different name). Live URL: https://fairytails123.github.io/frmdisplay/.

To deploy a change to `Feedingreport_display.html`:
1. `gh repo clone Fairytails123/frmdisplay` into a temp dir.
2. Write this file's content over the repo's `index.html`, **CR-stripped** (`sed 's/\r$//'`) so the repo stays LF and the diff is just your real change.
3. **Set a git identity on the fresh clone** — it has none by default (`git config user.name "Fairytails123"; git config user.email "Fairytails123@users.noreply.github.com"`), or the commit fails with "Author identity unknown".
4. `git add index.html && git commit && git push`. GitHub Pages serves it in ~1 min (cache-bust with `?cb=<ts>`).
- **Network ops (gh/git clone, push, curl) require the Bash tool's sandbox disabled.**
- Keep this folder's file and the repo's `index.html` in sync — edits made only in OneDrive don't reach the TV until pushed, and vice-versa.

## Verifying changes (no build/test tooling)

Same de-facto test step as the Feeding Manager project: a **throwaway headless Node harness** that loads the **actual** inline `<script>` (extract it, run via Node's `vm`/`new Function` with `document`, `fetch`, timers, etc. stubbed) and exercises pure logic without a browser. The script only registers a `DOMContentLoaded` handler at load (and starts `setInterval`s inside it), so a stub `document.addEventListener` that doesn't fire keeps the load side-effect-free. Example proven this way (2026-06-03): call `applyData({ dogs: [scrambled positions], … })` and assert each `pens[penId]` comes out ordered by `position`, with missing positions preserving server order. Verify before pushing — the TV is a live, visible surface.

## Conventions / gotchas

- **Read-only by design.** Never add a write/POST path here; all mutations belong to the tablet + GAS in the Feeding Manager project.
- **Pen IDs and status strings are a shared contract** with that project (above) — don't diverge them here unilaterally.
- The `APPS_SCRIPT_URL` is the single integration point. It is a public Pages-hosted constant (the same web-app URL the tablet uses); treat a blank display as "wrong/changed URL or GAS is erroring" before suspecting the render code.
