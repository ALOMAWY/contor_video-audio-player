# Contor

**Contor** is a cross-platform video and audio player built on one shared playback core. The same
state machine, time math, source resolution, and queue logic drive three shells:

| Shell | Stack | Package output |
| --- | --- | --- |
| Windows desktop | Electron + React | NSIS installer + portable `.exe` |
| Android | React Native / Expo (ExoPlayer) | release `.apk` |
| Web | React + Vite | static `dist/` |

Paste a link or drop a file and the unified control surface behaves identically everywhere: seek by
seconds, precise speed control, chapters, queue with autoplay, A–B loop, subtitles, PiP, and
fullscreen.

---

## Repository layout

```
contor/
├── apps/
│   ├── web-react/            # React + Vite renderer (also the Electron renderer)
│   ├── desktop-electron/     # Electron main/preload + electron-builder packaging
│   └── mobile-native/        # Expo / React Native Android app + mic-monitor native module
├── packages/
│   ├── core-player/          # Shared playback state machine, source resolver, chapters, queue
│   ├── media-library/        # Library state, folder/playlist grouping, tags, thumbnails
│   ├── shared-types/         # Source, provider, chapter, and capability contracts
│   └── shared-ui-tokens/     # Solid color tokens and spacing scale
├── images/                   # Served by Vite as publicDir (logo + favicons)
└── scripts/                  # Icon generation and build helpers
```

`packages/*` hold every piece of logic that must not diverge between platforms. The platform shells
only render state and forward user intent; they never re-implement playback rules.

## Commands

```bash
npm install                 # install workspace dependencies
npm run dev:web             # Vite dev server for the web renderer
npm run typecheck           # tsc --noEmit across all 7 workspaces
npm run build               # turbo build (web bundle, then Electron packaging)
npm run build:win           # Electron NSIS installer + portable .exe
npm run build:android       # Expo prebuild + Gradle assembleRelease
```

Node 20+ and npm 10 are expected (`packageManager` is pinned to `npm@10.9.3`).

## Architecture notes

**Source resolution.** `resolveSource()` classifies a local path or a direct media URL into the
`VideoSource` contract. `resolvePlaybackInput()` classifies a pasted *page* link into
`ResolvedPlayback` with a `PlaybackMode` and a `PlaybackCapabilities` set. Both live in
`@contor/core-player`; no UI package inspects provider URL formats.

**Playback modes.** `local`, `direct`, `hls`, `dash`, `provider-embed`, `telegram-client`,
`external-fallback`. A provider page URL is never handed to a native media element, and an
unsupported page degrades to `external-fallback` instead of a broken player.

**Streaming.** HLS uses `hls.js`, DASH uses `dash.js`, and progressive links use the native
`HTMLMediaElement`. Android uses `react-native-video`/ExoPlayer for every app-controlled mode.

**Security posture.** The Electron main process runs with `contextIsolation: true`,
`nodeIntegration: false`, `sandbox: true`, and `webSecurity: true`, denies all window opens, and
intercepts navigation away from the bundled renderer.

## Design rules

These are fixed constraints, not preferences:

- **No gradients.** Solid or translucent solid colors only. No `linear-gradient`, no
  `radial-gradient`, no multi-stop backgrounds in any stylesheet or icon.
- **Nothing non-essential is visible by default.** Quality, subtitles, audio tracks, settings, and
  provider metadata stay hidden until the user asks for them.
- **Icons, not text labels,** for playback actions — always with an accessible name and a tooltip.
- **A control the source cannot honor is hidden,** not rendered as an inert button.
- Classic YouTube-like identity: `#0F0F0F` dark / `#FFFFFF` light with switchable solid accents.
  The UI is Arabic and laid out RTL.

### Audit finding — the no-gradient rule was being violated

An audit on 2026-09-27 found **20 gradient declarations across 9 files** in the web renderer, which
breaks the first mandatory design rule and a release gate:

| File | Declarations |
| --- | --- |
| `apps/web-react/src/screens/SettingsScreen.tsx` | 5 |
| `apps/web-react/src/screens/LibraryScreen.tsx` | 4 |
| `apps/web-react/src/screens/AudioRunner.tsx` | 2 (one `bg-gradient-to-br`, one two-layer `radial-gradient`) |
| `apps/web-react/src/screens/PlayerScreen.tsx` | 2 |
| `apps/web-react/src/screens/DetailsScreen.tsx` | 2 |
| `apps/web-react/src/ui.tsx` | 2 |
| `apps/web-react/src/App.tsx` | 1 |
| `packages/media-library/src/thumbnail.ts` | 2 (`linear-gradient` placeholder strings) |

A grep for `gradient` over `apps/` and `packages/` is the regression check, and it must return
nothing. The UI stylesheets themselves (`styles.css`, `theme.css`) are already clean.
- **Icons, not text labels,** for playback actions — always with an accessible name and a tooltip.
- **A control the source cannot honor is hidden,** not rendered as an inert button.
- Classic YouTube-like identity: `#0F0F0F` dark / `#FFFFFF` light with switchable solid accents.
  The UI is Arabic and laid out RTL.

## Project status

| Area | State |
| --- | --- |
| Shared playback core, queue, chapters, settings | Implemented |
| Web renderer + Electron shell | Builds and runs |
| Android shell | Typechecks; release build requires the Android SDK |
| Provider page links (`provider-embed`) | Implemented for the documented host set |
| Authorized Telegram playback | Contract only — needs the TDLib client and an app `api_id`/`api_hash` |
| Contract test suite | Not yet written; test files are untracked by policy (see below) |

Provider behavior is intentionally conservative: Contor uses official embed surfaces and never
scrapes, proxies, caches, or extracts undocumented media URLs. Private, age-restricted, and
region-restricted content fails with a specific message rather than a workaround.

---

## Repository hygiene

**Test files, internal docs, and every `.env` variant are ignored on purpose** and never committed.
The tracked tree is source, build configuration, and this README.

What `.gitignore` excludes, and why:

| Category | Examples | Reason |
| --- | --- | --- |
| Dependencies & caches | `node_modules/`, `.turbo/`, `.vite/`, `*.tsbuildinfo` | Reproduced by `npm install` |
| Build output | `dist/`, `build/`, `out/`, `release/` | Generated; the Electron `release/` folder alone held four packaged binaries |
| **Environment & secrets** | `.env`, `.env.*`, `*.pem`, `*.jks`, `*.keystore`, `service-account*.json`, `google-services.json` | Only `.env.example` is tracked; a leaked provider key is unrecoverable |
| **Tests & fixtures** | `test/`, `__tests__/`, `*.test.*`, `*.spec.*`, `coverage/`, `fixtures/`, `test-*.mp4` | Provider fixtures must be licensed test content, and test media is large binary working material |
| **Internal docs** | `docs/`, `*.pdf`, `*.docx`, `notes/` | Requirement and review documents are internal; the public contract is this README |
| Evidence & recordings | `screenshots/`, `evidence/`, `*.har`, `*.snap` | Redacted evidence bundles are release artifacts, not source |
| Design source files | `*.psd`, `*.ai`, `*.sketch`, `*.blend` | Editable sources; the exported assets that ship are tracked |
| Logs & crash dumps | `*.log`, `*.pid`, `*.seed`, `crash*.log` | Machine-local noise |
| Android / Expo generated | `apps/mobile-native/android/`, `.expo/`, `*.apk`, `*.aab` | Regenerated by `expo prebuild` |
| Editor & OS | `.DS_Store`, `Thumbs.db`, `.idea/`, `.vscode/` | Not project content |

Two deliberate exceptions:

- **`images/` is tracked.** Vite serves it as `publicDir`, so `contor_logo.svg` and the favicon it
  backs are runtime app assets. Ignoring them would break the logo and favicon on a clean checkout.
- **`README.md` is tracked**, even though `*.md` under `docs/` is not. It is the public contract.

Because test files are untracked, the checks that matter are re-runnable from this file rather than
from a committed suite — see **Verification** below.

## Verification

Every change in this repository was validated before it was committed:

```bash
npm run typecheck     # 7/7 workspaces pass (core, media-library, shared-types,
                      # shared-ui-tokens, web, electron, mobile)
npm run build         # web production bundle succeeds
```

Browser behavior was checked against a running dev server with the Chrome DevTools MCP server:
initial render, console cleanliness, the accessibility tree, and the no-gradient rule. Findings and
fixes from those passes are recorded in the repository history.
