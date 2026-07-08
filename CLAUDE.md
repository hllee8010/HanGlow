# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

HanGlow is a mobile-first Korean language learning web app (Duolingo-style: lessons, quizzes, XP, streaks, premium paywall). The entire application lives in a single file, `index.html` (~7,700 lines) — HTML, CSS, and vanilla JavaScript together. There is no build system, no package.json, no tests, and no linter.

## Running the app

Open `index.html` directly in a browser, or serve it statically:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

There is nothing to build or install. The only external runtime dependencies are the Supabase JS client (loaded from jsDelivr CDN) and Google Fonts; mascot images are embedded as data URIs.

## Layout of index.html

Because everything is in one file, navigate by searching for function names, screen IDs, or the `═══`-style section banner comments rather than by line number.

- **Lines ~10–1430**: single `<style>` block. Design tokens are CSS custom properties in `:root` — the Dancheong (단청) 5-color palette (`--dc-red`, `--dc-blue`, `--dc-yellow`, `--dc-green`, `--dc-white`), a 4pt spacing grid, an 8-step type scale, and radius/shadow scales. Legacy aliases (`--hg-*`, `--main-blue`, etc.) point at the unified tokens; prefer the `--dc-*` / `--sp-*` / `--fs-*` tokens for new styles.
- **~1440–2488**: HTML for all screens. Each screen is a `<div class="screen" id="screen-...">`; navigation just toggles the `active` class.
- **~2489–7681**: single `<script>` block with all application JS.

## Architecture

### Screens and navigation

`showScreen(id)` is the router: it deactivates all `.screen` elements, activates the target, syncs the bottom nav highlight, and runs per-screen init hooks (`initHome`, `initMyScreen`, bookmark injection on detail screens). Screens: `screen-login`, `screen-home`, `screen-my`, `screen-subscribe`, `screen-diagnostic`, and six content sections — `phonics`, `convo`, `kculture`, `quiz`, `grammar`, `emotion` — each with a list screen and a `-detail` screen.

### Content data

All lesson content is hardcoded in large JS array constants, one per section: `phonicsDays`, `grammarDays`, `convoDays`, `kcultureDays`, `quizDays`, `emotionDays`, plus `diagQuestions` and `dailyWords`. Each day object carries `{day, level, title, titleKo, free, ...}` where `level` is `beginner` (Day 1–10), `intermediate` (11–20), or `advanced` (21–30). Each section has its own parallel set of render functions (`renderDayList`/`switchLevel` for phonics, `renderGrammarDayList`/`switchGrammarLevel` for grammar, etc.) — when changing shared behavior like day-card rendering or locking, check every section's copy.

### Premium gating

Days without `free:true` are locked; clicking a locked card calls `showPremium()` (the subscribe sheet) instead of the detail view. Roughly the first 3 days per section are free (`FREE_DAYS = 3`). There is no real payment flow — the paywall is UI only.

### State: localStorage first, Supabase second

All state is read/written to localStorage synchronously, then mirrored to Supabase with a ~1.5s debounce. The pattern is `load*/save*` (local) + `pull*FromCloud`/`schedulePush*`/`_push*Now` (cloud) per data type:

| Data | localStorage key | Supabase table |
|---|---|---|
| Progress (XP, streak, completed days, quiz scores, daily goal) | `hanglow_progress` | `progress` |
| Profile (name, etc.) | `hg_profile_v1` | `profiles` |
| Bookmarks | `hg_bookmarks_v1` | `bookmarks` |
| Wrong-answer notes | `hg_wrongnotes_v1` | `wrong_notes` |
| Diagnostic test result | `hg_diagnostic_v1` | `diagnostics` |
| Login mode / session flag | `hanglow_loggedIn` | — |
| TTS speed, onboarding seen, diag card dismissed | `hg_tts_rate_v1`, `hg_onboarded_v1`, `hg_diag_dismissed_v1` | — |

`saveProgress()` automatically calls `schedulePushProgress()` — always go through `loadProgress()`/`saveProgress()` rather than touching `localStorage` directly for progress data.

### Auth and offline behavior

Supabase client `sb` is created from the hardcoded URL/anon key near the top of the script; `SB_READY` guards every cloud call. Login modes: **Guest** (local-only, no backend), **email/password**, and **Google OAuth** via Supabase Auth; Apple/Amazon buttons fall back to guest. The app must keep working fully with no backend — any new cloud feature needs a local-only fallback behind `SB_READY`.

### Gamification

XP constants (`XP_LESSON`, `XP_QUIZ_CORRECT`, `XP_STREAK_BONUS`, `XP_GOAL_BONUS`), 10 level badges in `LEVELS`, a day-based streak (`updateStreak`), and a daily goal of `DAILY_GOAL = 2` lessons. Award XP through `addXP()` (it handles level-up toasts) and lesson completion through `incrementDailyGoal()`.

### Audio

Korean pronunciation uses the browser Web Speech API (`SpeechSynthesisUtterance` with `lang:'ko-KR'`), wrapped in `speakKorean()`. Playback rate comes from `getTTSRate()`.

## Conventions

- The UI targets a phone viewport (`body { max-width:430px }`); English UI copy with Korean lesson content. Fonts: Nunito + Noto Sans KR.
- No framework, no modules — everything is global functions in one script. Guard cross-references with `typeof fn === 'function'` checks as the existing code does, since declaration order matters.
- Escape any user- or data-derived strings inserted via `innerHTML` with the existing `escapeHtml()` helper.
