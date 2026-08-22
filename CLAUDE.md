# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Entrenador Musical" is a single-page music-practice trainer: note recognition, intervals,
octaves, and major/minor chords (root position + 2 inversions), for piano (via Web MIDI) or
guitar (SVG fretboard/chord diagrams). It also has a metronome mode, a routine/playlist builder,
multiple local profiles on one browser, and a Firebase-backed "Comunidad" screen (live presence,
star ranking, activity feed). All UI text and code comments are in Spanish — keep new code/comments
in Spanish to match.

There is no build system, no package.json, no test suite. The entire app is `index.html`
(~5300 lines: `<style>`, then markup, then one big inline `<script>` at the bottom). `404.html` is
Firebase's default not-found page (untouched boilerplate).

## Commands

- **Run locally**: just open `index.html` in a browser, or `firebase serve` / `firebase emulators:start --only hosting` for something closer to prod (needed for correct relative paths / no build step either way).
- **Deploy**: `firebase deploy --only hosting` from this folder. Project id is `entrenador-musical` (see `.firebaserc`), hosting site `entrenador-musical` (see `firebase.json`).
- No lint/build/test commands exist. Verify changes by opening the file in a browser and exercising the flow by hand (there's no headless test harness for the MIDI/audio/Firebase-dependent behavior).

## Architecture (all inside `index.html`)

### Script layout
The inline `<script>` starting ~line 1712 is organized as a flat sequence of `// ---- Section ----`
comment banners (search for `// ----` to get a table of contents) rather than modules/classes.
Roughly, top to bottom:
1. Static data tables: note spellings, `EXERCISES` (~line 1749), chord circle/root data, guitar
   tuning + chord shapes + fretboard levels, `INTERVALS` data.
2. Profiles + Firebase setup (auth, pending-account sync queue, presence).
3. Local settings/state (help mode, metronome, playlists, sound synthesis via Web Audio).
4. DOM refs (a long flat list of `const x = document.getElementById(...)`).
5. Persistence helpers (records/best-times, per-note adaptive stats, star progress).
6. Render/build functions (note picker, piano, guitar diagrams, records table).
7. MIDI setup, menu/settings interactions, theme/appearance.
8. Screen management + session/exercise engine (the "router").
9. Metronome engine, playlist ("rutina") builder + runner, Comunidad screen.
10. `// ---------- Init ----------` at the bottom: loads profiles and either shows the menu or the
    profile-select screen.

### Exercises
`EXERCISES` (line ~1749) is the single source of truth for exercise types: each entry has an `id`,
labels, a `modes` list (e.g. hand/zone), and `bands` describing the valid MIDI pitch range(s) per
mode. Special exercise behavior is flagged inline rather than via separate types/classes:
`isChords`, `isIntervals` flags on an exercise object switch the shared session engine into
chord-sequence or interval-quiz logic. `showNextTarget()` (~line 3979) is the router that dispatches
on these flags to `showNextChordTarget` / `showNextIntervalTarget` / `showNextNoteTarget`.

### Screens
Pure show/hide of sibling `<div>`s — `showScreen(name)` (~line 3897) hides every screen div then
un-hides the one matching `name` (`'menu'`, `'settings'`, `'exercise'`, `'countdown'`, `'session'`,
`'results'`, `'playlists'`, `'playlist-summary'`, `'profile-select'`, `'family'`). There's no router
library or URL state — navigation is just calling `showScreen()`.

### Profiles and storage namespacing
Multiple local profiles share one browser via key prefixing: `pkey(key)` (~line 2396) prepends
`p_<activeProfileId>_` to any storage key, and `storageGet/Set/Remove` always go through it. A
handful of keys are intentionally *not* namespaced (the profile list itself and which profile is
active — `STORAGE_PROFILES_KEY`, `STORAGE_ACTIVE_PROFILE_KEY`) since they must be readable before a
profile is chosen. When adding new persisted state, decide up front whether it's per-profile (go
through `pkey`/`storageGet`/`storageSet`) or global.

### Firebase (accounts, presence, Comunidad)
Loaded via the Firebase **compat** SDKs (v10.12.0, from gstatic — see script tags ~line 1207), not
the modular SDK, so the codebase uses `firebase.initializeApp(...)`, `firebase.auth()`,
`firebase.database()` global-namespace style throughout. `firebaseConfig` and all Firebase logic
live inline (~line 2287 onward). Key pieces:
- A local profile becomes a real Firebase account (email + 6-digit PIN as the password) via
  `createFirebaseAccount`; if offline, the attempt is queued in `STORAGE_PENDING_SYNC_KEY` and
  retried on the `online` event (`trySyncPendingAccounts`).
- Presence: `connectPresence()`/`disconnectPresence()` write to `global/presence/<uid>` in Realtime
  Database, using the `.info/connected` + `onDisconnect()` pattern (register the auto-remove
  *before* writing presence, to avoid a stale disconnect clobbering a fresh session).
- The Comunidad screen (~line 5206, `// ---------- Pantalla "Comunidad" (global) ----------`) reads
  presence, a star ranking, and an activity feed of completed routines from the shared Realtime
  Database tree — everyone shares the same global space (no per-user visibility rules beyond what's
  described here), so treat any new writes there as public data.

### Session/exercise engine
Session state is a set of module-level `let` globals (`sessionActive`, `targetPC`, `score`,
`reactionLog`, `multiProgress`, `chordProgress`, etc.) rather than an object/class — functions read
and mutate these directly. `showNextTarget()` picks the next prompt; `randomTarget()` +
`weightedPick()` implement adaptive selection (avoids repeating the previous note/its neighbors,
weights notes by historical performance when `adaptiveMode` is on). Records are tracked per
exercise-mode + duration via `saveBestIfRecord` / `saveBestScoreIfRecord` (~line 2832/2930), and a
"new record" is what currently drives the results-screen sparkle effect (`playRecordSparkle`) — this
is the hook point if a "new record" event needs to feed into the Comunidad activity feed too.

### Playlists ("rutinas")
A playlist is `{id, name, advanceMode, blocks:[{id, exerciseId, label, config, repetitions}]}`,
persisted under `STORAGE_PLAYLISTS_KEY`. Running one is driven by `playlistRunState` (queue of
blocks + per-block results) and the engine at `// ---------- Motor que corre una rutina completa
----------` (~line 5445). Run history is kept separately per playlist
(`STORAGE_PLAYLIST_HISTORY_KEY`) to support "compare to yesterday/last week/last month" views, and
an in-progress run can be persisted (`STORAGE_PLAYLIST_RUN_KEY`) so it survives a reload.

### Guitar mode
Guitar fretboard levels, chord shapes (`GUITAR_CHORD_SHAPES`), and note-position lookup
(`findGuitarNotePosition`) live around line 1970–2200. Guitar and piano share the same exercise
engine and MIDI-independent metronome input mode — `instrument` is a global setting, and guitar mode
always forces `effectiveInputMode` to the metronome (no MIDI guitar input).

## Comunidad activity feed
`pushFamilyActivity(label, score)` (~line 5216) is called from `buildResults()` (~line 4613) for
every completed standalone session (both the normal scored path and the metronome
"objetivos practicados" path), not just for completed routines — every session posts to
`global/activity`, always, regardless of whether it set a new record. Routines still push once as a
whole (`advancePlaylist()`, ~line 5529) rather than once per block.
