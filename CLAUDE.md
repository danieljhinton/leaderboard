# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A conference-booth scoreboard app for Halo: a registration kiosk (new/existing player sign-in + score entry) and a live leaderboard display, implemented as a **single published Claude Artifact** (`index.html`) rather than a conventional web app. There is no build step, no package manager, and no test suite — `index.html` is the entire product, edited directly and published straight to the live URL.

## Commands

There is nothing to install, build, lint, or test. The only "commands" that matter:

- **Publish a change**: after editing `index.html`, republish it with the Artifact tool passing `url: https://claude.ai/code/artifact/c28f7da9-79c7-4785-8628-64ac6e5da67c` (this republishes the *same* live link; omitting `url` creates a brand new, separate artifact) and `capabilities: {"db": {}, "downloads": true}` (capabilities are set at publish time via the tool, not in the HTML — omitting the `capabilities` argument on a republish silently carries the previous declaration forward, but it's safer to always pass it explicitly here since this project relies on it).
- **Commit and push**: standard git against the `leaderboard` GitHub repo (`danieljhinton/leaderboard`) — this repo is a source backup/history of the artifact, not something that gets deployed from independently.
- **Local preview**: opening `index.html` directly in a browser will render the static layout, but every dynamic feature (registration, scores, leaderboard, CSV export) is inert — they all depend on `window.claude.use(...)`, which only exists inside the published Claude Artifact runtime.

## Architecture

Everything lives in one file: a `<title>`/`<link>`(Google Fonts)/`<style>` block, a single `<div id="app">`, and one `<script>` IIFE at the bottom. There is no framework.

**Render pattern.** A single `state` object drives everything. `render()` rebuilds `app.innerHTML` from scratch via string-templated HTML functions (`kioskHTML()`, `leaderboardHTML()`, `modeSelectHTML()`, etc.) — there is no diffing. Events are handled by three delegated listeners on `#app` (`click`, `submit`, `input`) that dispatch on a `data-action` attribute rather than per-element handlers, so adding a new interactive element means adding a `data-action` value and a `case` in the relevant switch. The one exception to "always full re-render" is the existing-player search box: its `input` handler calls `renderExistingListOnly()` to patch just the `#existing-list` container, because a full re-render would blow away the input's focus/cursor on every keystroke.

**Two device modes, one file.** The same artifact serves two different physical screens. On load, `init()` picks a mode from a `?mode=kiosk|leaderboard` URL param or from `localStorage["boothScoreboardMode"]`; if neither is set, it shows a mode-select screen so each device "remembers" what it's for. There's no auth distinction between modes — it's purely a UI/URL convention for the booth staff setting up devices.

**Kiosk step state machine** (`state.step`): `start` → `new` (registration form) or `existing` (searchable player list) → `score` (entry) → `done` (confirmation), driven by `goStep()`. `new` creates a player doc; `existing` selects one from a live query. Both converge on `score`.

**Data model** lives in the Claude Artifact `db` capability (a realtime, shared document store scoped to this one artifact — see `await claude.use("db")` calls):
- `players/{id}` — `name`, `company`, `email`, `emailLower` (lowercased, for duplicate-email lookups), `bestScore`, `lastScore`, `createdAt`, `updatedAt`. A player is one doc for their whole booth history; scoring never creates new rows — `submitScore()` always `update()`s the existing doc, raising `bestScore` only if the new score beats it.
- `config/event` — single doc holding the editable `title`/`subtitle` shown on both screens (set via the gear-icon settings modal).
- The leaderboard view queries `players` with `where("bestScore", ">=", 0)` — a player with `bestScore: null` (registered but never played) simply doesn't appear yet.
- "Remove from leaderboard" (from the leaderboard's click-to-edit modal) clears `bestScore`/`lastScore` back to `null` rather than deleting the player doc, so they drop off the board without losing their registration.

**Capabilities used** (declared only at publish time, see Commands above): `db` for all of the above, and `downloads` for the CSV export in the settings modal (`exportPlayersCsv()` — reads all `players` once, doesn't use the live leaderboard query, so it includes unscored registrants too).

**Fullscreen.** Because the artifact renders inside an embedded frame on claude.ai, OS/browser fullscreen (F11) only fullscreens the outer page chrome. `toggleFullscreen()` calls the Fullscreen API directly on `document.documentElement` from inside the artifact so just its content fills the screen; this can be blocked by the embedding frame's permissions, which is handled as a non-fatal `state.fullscreenBlocked` notice rather than an error.

**Brand system.** Colors, radii, and the gold-medal/rank treatment follow Halo's internal brand book (Halo Blue `#00CEFF`, Navy `#002D5B`, Midnight `#051830`, plus Teal/Sky/Lilac/Peach as supporting tones — see the `:root` custom properties). The brand's actual typeface, Centra No2, is a commercial font unavailable via the CDNs artifacts can load from; Poppins is used as a same-weight (400/500/700) substitute throughout. Light/dark mode is handled per Claude Artifact conventions (`:root`, `prefers-color-scheme`, `[data-theme]`) for the kiosk; the leaderboard is an intentional single (dark/"Midnight") theme regardless of viewer setting, matching how the brand book uses dark backgrounds for hero/display moments.
