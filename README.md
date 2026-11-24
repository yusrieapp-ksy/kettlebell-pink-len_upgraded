# KettlebellPink

A 32:00 full-body kettlebell workout app with a guided voice-coached timer, built as an installable Progressive Web App (PWA). Single-file architecture: the entire app lives in `index.html` (~136 KB, ~2,300 lines), with a small service worker for offline caching and update notifications.

**Live app:** `https://<your-username>.github.io/<repo-name>/` (after hosting — see below)

---

## 1. Files in this project

| File | Size | Required for hosting? | Purpose |
|---|---|---|---|
| `index.html` | ~136 KB | **Yes** | The entire app — markup, CSS, and all JavaScript (workout data, timer engine, voice coach, audio, animations, stats, PWA install logic). |
| `sw.js` | ~1.1 KB | **Yes** | Service worker. Caches app files for offline use and powers the "App updated" notification. |
| `manifest.webmanifest` | ~0.6 KB | **Yes** | PWA manifest (name, icons, colors, start URL) — enables "Add to Home Screen" / install prompts. Note: `index.html` also regenerates this at runtime with inline data-URL icons (see §5.7); the static file is the initial/fallback reference the `<link>` tag points to before JS runs. |
| `icon-192.png` | ~21 KB | Yes | App icon, 192×192. |
| `icon-512.png` | ~60 KB | Yes | App icon, 512×512. |
| `icon-maskable-512.png` | ~47 KB | Yes | Maskable variant (512×512) for adaptive icon shapes on Android. |
| `apple-touch-icon.png` | ~20 KB | Yes | iOS home-screen icon (180×180 referenced in `<link rel="apple-touch-icon">`). |
| `favicon.png` | ~8 KB | Yes | Browser tab icon. |
| `README.md` | this file | No | Documentation only — does not affect the running app. |

**All 8 non-README files must sit together in the same folder** (the repo root, or the root of whichever folder GitHub Pages serves) — the app references them with relative paths (`./`, `index.html`, `manifest.webmanifest`, the icon filenames), so they cannot be nested in subfolders without edits.

No build step, no `node_modules`, no bundler. You can open `index.html` directly in a browser for local testing, though the service worker and install prompt only activate when served over `http(s)://` (see §3).


---

## 2. Hosting on GitHub Pages

1. Create a new **public** GitHub repository (e.g. `kettlebellpink`).
2. Upload all 8 files from §1 (everything except README is required) to the **repository root** — do not put them in a subfolder like `/docs` unless you also update Pages settings to match.
3. In the repo: **Settings → Pages → Build and deployment → Source: "Deploy from a branch" → Branch: `main` / `(root)` → Save.**
4. Wait 1-2 minutes. Your app will be live at:
   `https://<your-username>.github.io/<repo-name>/`
5. Open that URL on your phone and install it (see §3).

### Updating after hosting
1. Push the changed files over the old ones (commit + push, or re-upload via GitHub's web UI).
2. **Always bump the cache name in `sw.js`** when `index.html` or any cached asset changes — e.g. `kettlebellpink-v6` → `kettlebellpink-v7`. This is what makes the browser notice there's a new version; without it, phones that already installed the app may keep running the old cached copy indefinitely.
3. Once a user's browser picks up the new service worker, they'll see an **"App updated"** toast with a **Refresh** button (see §5.9) — it will not auto-reload mid-workout.

---

## 3. Installing the app (for end users)

- **iPhone/iPad:** Open the link in **Safari** (must be Safari, not Chrome) → tap the **Share** icon → **Add to Home Screen** → **Add**. Launch from the pink kettlebell icon on the Home Screen — it opens full-screen, no browser chrome.
- **Android:** Open the link in **Chrome** → menu (⋮) → **Install app**. Or use the in-app install button (top-right toolbar icon) which calls the native `beforeinstallprompt` flow automatically.
- **Desktop Chrome/Edge:** An install icon appears in the address bar, or use the in-app install button.


---

## 4. Architecture overview

### 4.1 Single-file design, shared global scope
Everything lives in `index.html`, split across **4 sequential `<script>` blocks** (approx. lines 351, 1801, 2008, 2150). They are **not** modules — all run in the same global scope, in document order, as the browser parses top to bottom. This is a deliberate pattern used throughout the app's history of incremental upgrades:

- **Block 1** (line 351): Core app — PWA install logic, workout data (`buildFullTimeline()`), the timer engine, audio/voice coach, animation renderer, and all base function definitions (`renderStep`, `tick`, `togglePlay`, `nextStep`, `advanceToNextStep`, etc.).
- **Block 2** (line 1801, "v3 upgrades"): Natural voice delivery, louder/ducking music, iOS audio-session handling, clearer on-screen instructions, a voice picker. **Overrides** several Block 1 functions by capturing the old one (`const _rs = renderStep;`) and reassigning `renderStep = function(...) { _rs(...); /* extra behavior */ }` — a wrap-chain, not a rewrite.
- **Block 3** (line 2008): Pose-animation engine data (`A`/`A2` keyframe tables) driving the stick-figure SVG demonstrations for each exercise.
- **Block 4** (line 2150, "v4"): Groovier phase-aware background music, get-ready countdown, session restore after the phone sleeps, catch-up logic, post-workout rating, and **lifetime stats** (persisted sessions/streak/time). Wraps `initAudio`, `syncAudioMixerUI`, `renderStep` (again), `tick`, `togglePlay`, `openCheckInModal`, `openCompletionSummary`.

**Why this matters for future edits:** 7 function names are intentionally defined twice (`initAudio`, `syncAudioMixerUI`, `renderStep`, `tick`, `togglePlay`, `openCheckInModal`, `openCompletionSummary`) — each later definition captures and calls the earlier one via a closure variable (e.g. `_rs`, `_tp`). This is correct, working code, **not dead code** — do not "clean up" by deleting the first definition. When adding new behavior to one of these functions, decide deliberately whether it belongs in the base definition or a new wrap added at the end of the file (wrapping is safer/more additive).

### 4.2 No frameworks, no dependencies
Vanilla HTML/CSS/JS only. No React/Vue, no npm packages, no CDN scripts except two Google Fonts `<link>` tags (`Plus Jakarta Sans`). All audio is synthesized at runtime via the Web Audio API (`playTone`, `playSoundCue`) — there are no `.mp3`/`.wav` asset files. All exercise "demo" visuals are a single inline `<svg>` with a JS-driven stick figure (`#fig`), not video or image sprites.


---

## 5. App features & data model

### 5.1 The workout
A fixed 32:00 session built by `buildFullTimeline()` (line 671) into a flat `TIMELINE` array of step objects (warm-up → 3 strength rounds → cool-down), each with a `type` (`work` / `rest` / `warmup` / `cooldown`), `duration`, `title`, `cue` (form guidance text), `voiceStart`/`voiceMid` (spoken coaching), `bell`/`bellType`/`bellSwap` (6 kg / 8 kg / bodyweight), and `anim` (which pose-animation keyframe set to play). Three phases:
- **Phase 1 — Dynamic Warm-Up**
- **Phase 2 — Round 1-3/3** (strength circuit, alternating work/rest, with water-break transitions between rounds)
- **Phase 3 — Decompression Cool-Down**

### 5.2 Timer engine
`tick()` runs once per second via `setInterval`, counting `remainingSec` down per step. At fixed thresholds it fires: halfway posture cues, a 20-second "up next" voice preview + visual flash (see §5.8), a spoken 10→1 countdown, tick chimes + haptics on 3-2-1, and side-switch cues on dual-sided exercises. Session state (current step, remaining time, completed steps, selected day/time) is saved to `localStorage` on every meaningful change and restored on reload (`kbp_session_v1`, see §5.6) so refreshing or reopening mid-workout doesn't lose progress.

### 5.3 Voice coach
Uses the browser's built-in Web Speech API (`speechSynthesis`). Picks a female voice automatically where available, with phrase-by-phrase delivery for natural pacing. Fully optional/toggleable — see the audio mixer (§5.5).

### 5.4 Pose-animation demo
The `A`/`A2` objects (Block 3) are keyframe tables keyed by exercise/round, each entry defining a duration (`d`) and a sequence of `(time, pose)` keyframes. `renderAnimFrame()` interpolates between them every frame to draw the stick figure (`#fig`) inside the SVG stage. `A` also drives the tempo-pill pulse dot's cadence (`--td` CSS variable) — **these two tables must stay numerically in sync** (a drift bug here was found and fixed in Nov 2026; see §7).

### 5.5 Audio mixer
A popover (`#audioMixerPopover`, top-right "speaker" button) with three independent toggles — Voice Coach, Chill Vibe Music, Countdown & Chimes — plus a music-volume cycle (Low/Med). Preferences persist in `localStorage` under `kbp_audio_prefs_v2`. All sound is generated live via Web Audio oscillators (`playTone`); there are no audio asset files.


### 5.6 Persistence — localStorage keys
| Key | Holds | Reset behavior |
|---|---|---|
| `kbp_audio_prefs_v2` | Voice/music/chimes on-off + music volume level | Never resets |
| `kbp_weekly_log_v2` | This ISO-week's logged workout days (Mon-Sun) + session times | Resets each new ISO week (or manually via "Start Fresh Week") |
| `kbp_wrap_pending` | A transient flag marking "Sunday wrap-up not yet shown" | Cleared once the wrap-up is shown; not a durable store |
| `kbp_lifetime_v1` | Total sessions completed, total training seconds, current streak, best streak — **persists across weekly resets** | Never resets (this is the point of keeping it separate from the weekly log) |
| `kbp_session_v1` | In-progress session snapshot (step index, remaining seconds, completed steps, day/time) for resume-after-reload, auto-discarded after 3 hours | Cleared on workout completion or after 3h staleness |

All reads/writes are wrapped in `try/catch` so the app degrades gracefully if `localStorage` is disabled (e.g. private browsing) — stats simply won't persist, but nothing throws.

### 5.7 PWA install & manifest
`index.html` dynamically regenerates the manifest at runtime (canvas-rendered PNG icons + an SVG fallback, assembled into a `Blob` URL that replaces the `<link id="pwaManifest">` href). The static `manifest.webmanifest` file on disk is the initial reference the `<link>` tag points to before this JS runs; keep it present and valid, but know that the *live* manifest content browsers actually use comes from the JS-generated blob. Install prompts: Android/Desktop Chrome listens for `beforeinstallprompt` and exposes an in-app "Install" button; iOS Safari has no such API, so the app shows manual "Add to Home Screen" instructions instead (`#installModal`).

### 5.8 Accessibility
All 4 modal dialogs (`timelineModal`, `installModal`, `completeModal`, `checkInModal`) carry `role="dialog"`, `aria-modal="true"`, and `aria-labelledby` pointing at their heading. The timer's countdown number is `aria-hidden` (would spam screen readers every second); the timer status text and exercise title are `aria-live="polite"` so meaningful state/step changes are announced without second-by-second noise. The "Up Next" text visually flashes (`.flash` class) at the same 20-second mark the voice coach announces it, giving sound-off users the same cue.

### 5.9 Service worker & update flow
`sw.js` precaches the 8 app files on install, serves cached copies offline, and always tries network-first for navigations (falling back to cache). `index.html` listens for `controllerchange` on `navigator.serviceWorker` and shows a persistent **"App updated"** toast with a **Refresh** button — it deliberately does **not** auto-reload, so an in-progress workout is never interrupted. Bumping the `CACHE` constant in `sw.js` (currently `kettlebellpink-v7`) is what triggers this flow for existing installs — see §2.


---

## 6. Design language (colors & motion)

Theme color swaps by workout phase via CSS custom properties on `<body data-t="...">`:
| Phase | `--ac` (accent) | Feel |
|---|---|---|
| Work (default) | `#EC4899` (pink) | Default brand pink |
| Warm-up | `#D946EF` (magenta) | |
| Rest | `#10B981` (green) | Deliberately shares the app's existing "done/success" green (used on completed week-strip days, the rest bell badge, the wake indicator) so rest is instantly recognizable as a distinct, calming state rather than a pink-adjacent coral |
| Cool-down | `#C084FC` (purple) | |

Motion is deliberately differentiated between work and rest: the ring's pulsing aura and the tempo-pill's pulse dot both run slower with a gentler scale swing during `data-t="rest"` than during active work, reinforcing "breathe" vs "push" without needing extra UI. Exercise-to-exercise transitions (title, variation label, and the demo stage) crossfade together on a single `requestAnimationFrame` tick rather than swapping instantly or independently.


---

## 7. Change history (chronological, most recent first)

> Kept here as a running log so future edits have context on *why* things are the way they are. Add a new entry at the top when you make further changes.

**Nov 2026 — Critical fix: workout wouldn't start** (`sw.js` → v7)
- **Bug fix:** tapping "Start Workout" froze on the "GET READY" countdown overlay at **5** and never advanced — the app was completely unusable. Root cause: `_wps` (words-per-second, used by the voice-timing helper `_est()`) was referenced and mutated in the v3-upgrades script block but **never declared anywhere** in the file. The very first countdown call, `speakCoach('Get ready. Five.')`, has 2 sentences, which routes through the budget-fitting branch that calls `_est()` — throwing an uncaught `ReferenceError` that silently aborted `togglePlay()` *before* the line that creates the countdown's `setInterval`. Fixed by declaring `let _wps = 2.5;` (a sensible default words/sec) right before its first use; the adaptive logic in `_vNext()` that refines it over time was already correct and untouched.
- Hardening: the "GET READY" overlay (`#leadIn`) had `pointer-events:none`, so any tap during the countdown silently leaked through to the Play/Prev/Next buttons underneath instead of being caught by the overlay's own cancel handling. Changed to `pointer-events:auto` only while `.on` (visible), and added an explicit click handler on the overlay that cancels the countdown cleanly (reusing the same cancel logic already in `togglePlay`) — this was not the root cause, but prevents a similar "nothing visibly happens" dead end in the future.

**Nov 2026 — Polish round 2** (`sw.js` → v6)
- Rest-phase rhythm: tempo dot now pulses 1.4x slower with a softer scale swing during rest, layered on the pre-existing slower aura.
- Unified crossfade: title + variation label + stage SVG now fade together on step change (previously: independent bounce-pop on title/label, separate pop on stage, unsynchronized).
- "Up Next" text visually flashes at the same 20s mark the voice coach announces it (sound-off parity).
- Completion climax: 4-note chime extended to 5 notes with a higher resolution + layered undertone; haptic pattern strengthened to an ascending crescendo; fixed a gap where the manual "Done"-button completion path had no haptic at all.
- Rest-phase accent color changed from coral (`#FB7185`, still pink-adjacent) to green (`#10B981`, reusing the existing "done" semantic color) for instant phase recognition.
- Week-strip empty-state dot redesigned from a filled "·" (read as a stray typo) to a dashed-outline ring; fixed a resulting bug where the dashed border would've persisted under the solid green checkmark on completed days.
- Header icon buttons (audio mixer, week badge, install) enlarged 38px → 44px (standard minimum touch target) for easier one-handed thumb reach; added a dedicated close button inside the audio mixer popover itself.
- Added `role="dialog"`/`aria-modal`/`aria-labelledby` to all 4 modals; `aria-live="polite"` on timer status text and exercise title; `aria-hidden` on the per-second countdown number.

**Nov 2026 — Polish round 1** (`sw.js` v4 → v5)
- Added persistent lifetime stats (`kbp_lifetime_v1`): total sessions, total training time, current streak, best streak — survives weekly resets. Hooked into `recordCompletedWorkout()`, the single trusted completion path. Streak dates use local-time components (not UTC `toISOString`) to avoid midnight-timezone errors.
- Added a red color-shift + one-shot scale pulse to the timer digits during the final 3 seconds of each interval.
- **Bug fix:** the tempo-pill pulse cadence (`A`) had drifted out of sync with the actual pose-animation duration (`A2`) on 5 of 17 exercises — most severely `cool_jefferson` (pill pulsing ~2x the real animation speed). Synced all 17 pairs exactly.
- Added a brief opacity fade to the demo figure (`#fig`) on step change (precursor to the full crossfade added in round 2).
- Slowed the ring's breathing-aura pulse during rest phases (precursor to the fuller rest/work rhythm differentiation in round 2).
- Added the service-worker update notification: a `controllerchange` listener shows a persistent "App updated" toast with a user-controlled Refresh button (no silent auto-reload, so an in-progress workout is never interrupted).

**Earlier — Cleanup**
- Removed 11 genuinely dead/duplicate function definitions that were silently overridden and never reachable (distinct from the intentional wrap-chain pattern described in §4.1). No behavior change; reduced file size by ~500 lines.
