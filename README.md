# EDU-RIVALS Client

Desktop companion for [EDU-RIVALS](https://edurivals.com) — the competitive collegiate esports platform. The client reads live data from the League of Legends client in real time and reports it to the EDU-RIVALS tournament system for automated match tracking, draft capture, and live spectator feeds.

**Latest release: [v0.7.1](https://github.com/Umopep1sdn/rivals-client-releases/releases/latest)**

> Windows 10/11 &middot; One-click installer &middot; Auto-updates in the background

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Tech Stack](#tech-stack)
- [Development](#development)
- [Building](#building)
- [Releases & Auto-Updates](#releases--auto-updates)
- [IPC API Reference](#ipc-api-reference)
- [Security](#security)
- [License](#license)

---

## Features

### League Client Integration
- **Automatic detection** of the running League client via process inspection or lockfile
- **Champion Select tracking** — bans, picks, summoner spells, and pick phase in real time
- **Live game data** — player stats, kills, CS, gold, items, objectives (dragons, barons, grubs, towers, inhibitors)
- **End-of-game results** — full scoreboard with KDA, CS, gold, items, runes, and damage
- **Rune & summoner spell display** with DDragon CDN assets

### Tournament System
- **Auto-detect** tournament matches by matching custom lobby members against scheduled brackets
- **Dual-reporter architecture** — one player reports full data, others validate for integrity
- **Leader election** — automatic priority-based leader selection with failover
- **Real-time sync** — Supabase Realtime channels coordinate all connected players
- **Series tracking** — automatic game number increment and series completion detection
- **Heartbeat monitoring** — detects disconnections and triggers re-election

### Desktop App
- **Custom frameless window** with a built-in title bar (minimize, maximize, close)
- **One-click NSIS installer** — installs per-machine with desktop and Start Menu shortcuts
- **Background auto-updates** — downloads new versions silently, prompts to restart when ready
- **Supabase authentication** — email/password sign-in with encrypted token storage
- **Onboarding flow** — profile setup and Riot account linking

---

## Installation

1. Download the latest `edu-rivals-client-setup.exe` from [Releases](https://github.com/Umopep1sdn/rivals-client-releases/releases/latest)
2. Run the installer — it installs to `Program Files` and creates shortcuts automatically
3. Launch **EDU-RIVALS Client** from the desktop or Start Menu
4. Sign in with your EDU-RIVALS account
5. Make sure League of Legends is running — the client auto-detects it

The client auto-updates in the background. When a new version is downloaded, you'll see a banner prompting you to restart.

---

## How It Works

```
League Client (LCU API)          EDU-RIVALS Client              Supabase Backend
━━━━━━━━━━━━━━━━━━━━━━━━    ━━━━━━━━━━━━━━━━━━━━━━━━━    ━━━━━━━━━━━━━━━━━━━━━━

  lockfile / process args        LcuStateMachine           Edge Function
  ──────────────────────► ┌─────────────────────┐    ┌──────────────────┐
  Port, password, pid     │ Poll every 1s/5s    │    │ lcu-match-import │
                          │                     │    │                  │
  /lol-champ-select/ ◄──►│ DISCONNECTED        │    │  Draft updates   │
  /lol-gameflow/     ◄──►│  → IDLE             │───►│  Game state      │
  /lol-summoner/     ◄──►│  → CHAMP_SELECT     │    │  Timeline events │
  /lol-end-of-game/  ◄──►│  → IN_GAME          │    │  Match results   │
  /allGameData (2999)◄──►│  → END_OF_GAME      │    │  Validation snaps│
                          └────────┬────────────┘    └──────────────────┘
                                   │                          ▲
                                   │ IPC Bridge               │ HTTPS POST
                                   ▼                          │
                          ┌─────────────────────┐    ┌──────────────────┐
                          │ React Renderer      │    │ DataReporter     │
                          │                     │    │ MatchSession     │
                          │ DraftView           │    │ MatchDetector    │
                          │ LiveGameView        │    │                  │
                          │ MatchResultView     │    │ Realtime channel │
                          │ TournamentMatchView │◄──►│ (player sync)   │
                          └─────────────────────┘    └──────────────────┘
```

### State Machine

The core of the client is `LcuStateMachine`, which polls the League client API and transitions through these states:

| State | Trigger | What Happens |
|---|---|---|
| `DISCONNECTED` | App launch / League closed | Polls every 5s looking for League process |
| `IDLE` | League client found | Connected, waiting for a game. Polls every 1s |
| `CHAMP_SELECT` | Champion select starts | Emits draft updates (bans, picks, spells) |
| `IN_GAME` | Game loads | Reads Live Client API on port 2999 for real-time stats |
| `END_OF_GAME` | Game ends | Captures final scoreboard, resets to IDLE |

### Tournament Flow

1. Player signs in and the client fetches today's scheduled brackets
2. Player creates a custom lobby in League
3. Client detects lobby members and matches PUUIDs against bracket team rosters
4. Auto-joins the detected match — opens a Supabase Realtime channel
5. Leader election runs (priority-based) — the leader becomes the **Reporter**
6. Other players become **Validators** (efficiency mode: throttled 5s update batches)
7. Reporter sends draft data, game state, events, and results to the edge function
8. Validators send periodic validation snapshots for integrity checking
9. After the game, series tracking increments the game number or completes the series

---

## Architecture

```
rivals-client/
├── Main Process (Electron / Node.js)
│   ├── Window management, IPC handler registration
│   ├── LCU integration (state machine, API client, data transforms)
│   ├── Tournament system (data reporter, match detection, session sync)
│   ├── Auth (Supabase client, encrypted token storage)
│   └── Auto-updater (electron-updater)
│
├── Preload (Context Bridge)
│   └── Exposes typed IPC API to renderer via window.electronAPI
│
└── Renderer (React / Vite)
    ├── Auth views (Login, Onboarding)
    ├── Game views (Draft, Live Game, Match Result)
    ├── Tournament views (Match status, Event log)
    └── Shell (Titlebar, Header, Update banner)
```

### Key Classes

| Class | File | Responsibility |
|---|---|---|
| `LcuStateMachine` | `src/main/lcu/lcu-state-machine.ts` | Polls LCU API, manages game state, emits events |
| `LcuClient` | `src/main/lcu/lcu-client.ts` | Credential detection (process/lockfile), HTTP requests to LCU |
| `ChampionDataManager` | `src/main/lcu/champion-data.ts` | DDragon API for champion/rune/item metadata |
| `DataReporter` | `src/main/tournament/data-reporter.ts` | POSTs game data to Supabase edge function |
| `MatchSession` | `src/main/tournament/match-session.ts` | Realtime channel, leader election, player sync |
| `MatchDetector` | `src/main/tournament/match-detector.ts` | Matches lobby composition to tournament brackets |
| `AuthManager` | `src/main/auth/auth-manager.ts` | Supabase auth, encrypted token persistence |

---

## Project Structure

```
rivals-client/
├── src/
│   ├── main/                         # Electron main process
│   │   ├── index.ts                  # Entry point, window creation, IPC handlers
│   │   ├── auto-updater.ts           # electron-updater integration
│   │   ├── auth/
│   │   │   ├── auth-manager.ts       # Supabase auth + encrypted token storage
│   │   │   └── supabase-client.ts    # Supabase client singleton
│   │   ├── lcu/
│   │   │   ├── lcu-client.ts         # LCU credential detection + HTTP wrappers
│   │   │   ├── lcu-state-machine.ts  # State machine (DISCONNECTED → END_OF_GAME)
│   │   │   ├── lcu-settings.ts       # League path persistence
│   │   │   ├── champion-data.ts      # DDragon champion/rune data
│   │   │   └── lcu-transforms.ts     # Raw API → typed data transforms
│   │   └── tournament/
│   │       ├── data-reporter.ts      # Edge function POST calls
│   │       ├── match-detector.ts     # Lobby-to-bracket matching
│   │       └── match-session.ts      # Realtime sync + leader election
│   ├── preload/
│   │   ├── index.ts                  # IPC bridge (contextBridge.exposeInMainWorld)
│   │   └── index.d.ts               # TypeScript declarations for window.electronAPI
│   └── renderer/src/
│       ├── App.tsx                   # Root component, state management
│       ├── main.tsx                  # React DOM entry
│       ├── types.ts                  # Shared interfaces
│       ├── ddragon.ts               # DDragon CDN helpers
│       └── components/
│           ├── Titlebar.tsx          # Custom window chrome
│           ├── Header.tsx            # Status bar + user context
│           ├── LoginView.tsx         # Auth form
│           ├── OnboardingView.tsx    # Profile + Riot account setup
│           ├── IdleView.tsx          # Waiting-for-game state
│           ├── DraftView.tsx         # Champion select visualization
│           ├── LiveGameView.tsx      # In-game stats + scoreboard
│           ├── MatchResultView.tsx   # End-of-game breakdown
│           ├── TournamentMatchView.tsx # Bracket + player connections
│           ├── EventLog.tsx          # Kill/event timeline panel
│           ├── MatchStatusBar.tsx    # Connected player count
│           ├── ChampionIcon.tsx      # Champion portrait component
│           ├── ItemIcon.tsx          # Item icon component
│           ├── RuneDisplay.tsx       # Rune tree display
│           └── ErrorBoundary.tsx     # React error fallback
├── resources/
│   ├── icon.png                     # App icon
│   └── icon.ico                     # Windows installer icon
├── package.json
├── electron.vite.config.ts          # Vite config (main + preload + renderer)
├── electron-builder.yml             # Installer & auto-update config
├── tailwind.config.js               # TailwindCSS theme
├── tsconfig.json                    # Root TS config
├── tsconfig.node.json               # Main/preload TS config
└── tsconfig.web.json                # Renderer TS config
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Electron 33 |
| Bundler | electron-vite (Vite 5) |
| UI | React 18, TypeScript 5.5 |
| Styling | TailwindCSS 3.4, PostCSS, custom dark theme |
| Icons | lucide-react |
| Backend | Supabase (Auth, Realtime, Edge Functions) |
| Persistence | electron-store (OS keychain encryption) |
| Auto-update | electron-updater (GitHub releases provider) |
| Installer | electron-builder (NSIS) |
| Compiler | SWC (via @vitejs/plugin-react-swc) |

---

## Development

### Prerequisites

- Node.js 18+
- npm 9+
- Windows 10/11 (for LCU integration testing)
- League of Legends installed (for live testing)

### Setup

```bash
git clone https://github.com/Umopep1sdn/rivals-client.git
cd rivals-client
npm install
```

### Run in dev mode

```bash
npm run dev
```

This launches the Electron app with hot-reload for the renderer. Main process changes require a restart.

### Environment

The Supabase URL and anon key are embedded in `src/main/auth/supabase-client.ts`. For development against a different Supabase project, update the values there.

---

## Building

### Windows installer

```bash
npm run build:win
```

Produces:
- `dist/edu-rivals-client-setup.exe` — one-click NSIS installer
- `dist/latest.yml` — auto-update manifest

### Other platforms (not actively maintained)

```bash
npm run build:mac    # macOS .dmg
npm run build:linux  # Linux .AppImage
```

---

## Releases & Auto-Updates

Releases are published to this repo (`Umopep1sdn/rivals-client-releases`) as GitHub Releases. The client uses `electron-updater` configured with:

```yaml
publish:
  provider: github
  owner: Umopep1sdn
  repo: rivals-client-releases
```

### Publishing a new release

1. Bump `version` in `package.json`
2. Run `npm run build:win`
3. Create a GitHub release on this repo with the version tag (e.g., `v0.7.1`)
4. Upload `edu-rivals-client-setup.exe` and `latest.yml` from `dist/`

Existing clients will detect the new version via `latest.yml`, download it in the background, and prompt the user to restart.

---

## IPC API Reference

The preload bridge exposes `window.electronAPI` with these namespaced channels:

### `lcu.*` — League Client

| Channel | Type | Description |
|---|---|---|
| `lcu:get-status` | invoke | Connection state, summoner name, game state, DDragon version |
| `lcu:start-watching` | invoke | Begin polling the League client |
| `lcu:stop-watching` | invoke | Stop polling |
| `lcu:get-league-path` | invoke | Get configured League install path |
| `lcu:set-league-path` | invoke | Set League install path manually |
| `lcu:browse-league-path` | invoke | Open native folder picker |
| `lcu:get-rune-data` | invoke | Fetch rune metadata by ID |
| `lcu:get-rune-tree-data` | invoke | Fetch full rune tree |
| `lcu:state-change` | event | State machine transition |
| `lcu:draft-update` | event | Champion select changes |
| `lcu:game-event` | event | In-game events (kills, objectives) |
| `lcu:game-event-batch` | event | Throttled event batch (validator mode) |
| `lcu:game-state` | event | Live scoreboard snapshot |
| `lcu:match-complete` | event | End-of-game stats |
| `lcu:lobby-update` | event | Custom lobby member changes |
| `lcu:spectate-detected` | event | Spectate mode detected |
| `lcu:account-mismatch` | event | LCU PUUID doesn't match verified account |

### `auth.*` — Authentication

| Channel | Type | Description |
|---|---|---|
| `auth:sign-in` | invoke | Email/password authentication |
| `auth:sign-out` | invoke | Sign out and clear tokens |
| `auth:get-session` | invoke | Check for active session |
| `auth:get-user` | invoke | Fetch user profile + game accounts |
| `auth:get-user-context` | invoke | Team name, display name |
| `auth:get-onboarding-status` | invoke | Profile completion status |
| `auth:state-change` | event | Auth state transitions |

### `tournament.*` — Tournament System

| Channel | Type | Description |
|---|---|---|
| `tournament:get-todays-matches` | invoke | Fetch scheduled brackets |
| `tournament:detect-match` | invoke | Match lobby to bracket |
| `tournament:join-match` | invoke | Enter tournament session |
| `tournament:get-match-status` | invoke | Connection state, role, player count |
| `tournament:set-ready` | invoke | Mark self as ready |
| `tournament:set-not-ready` | invoke | Unmark ready |
| `tournament:leave-match` | invoke | Leave session |
| `tournament:match-ready` | event | All players ready |
| `tournament:leader-changed` | event | New leader elected |
| `tournament:role-changed` | event | Reporter/Validator role assigned |
| `tournament:efficiency-mode` | event | Validator throttling toggled |
| `tournament:ready-changed` | event | Player readiness update |
| `tournament:connections-changed` | event | Player connected/disconnected |
| `tournament:series-complete` | event | Series winner determined |
| `tournament:game-complete` | event | Single game finished |

### `window.*` / `app.*` — Desktop

| Channel | Type | Description |
|---|---|---|
| `window:minimize` | send | Minimize window |
| `window:maximize` | send | Toggle maximize |
| `window:close` | send | Close app |
| `window:is-maximized` | invoke | Query maximized state |
| `window:get-version` | invoke | App version string |
| `window:maximized-change` | event | Maximize state changed |
| `app:update-downloading` | event | Update download progress |
| `app:update-ready` | event | Update ready to install |
| `app:install-update` | send | Quit and install update |
| `app:open-external` | invoke | Open URL in default browser |

---

## Security

- **Sandbox enabled** — renderer runs in a sandboxed process
- **Context isolation** — no direct Node.js access from renderer
- **Node integration disabled** — all main-process access goes through the typed IPC bridge
- **Web security enabled** — standard Chromium security policies
- **Token encryption** — auth tokens encrypted via OS keychain (electron-store + safeStorage)
- **Input validation** — all IPC handlers validate incoming data
- **URL allowlist** — `app:open-external` only permits HTTP/HTTPS URLs
- **Path validation** — League path inputs reject null bytes, traversal sequences, and relative paths
- **Self-signed cert handling** — LCU API uses a self-signed certificate; the client accepts it only for `127.0.0.1`

---

## License

Proprietary. All rights reserved by EDU-RIVALS.
