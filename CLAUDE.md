# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies (use npm, not pnpm/yarn)
npm install

# Development — starts Vite frontend + Rust backend together
npm run tauri dev

# Frontend only (port 1420)
npm run dev

# Tauri shell only (connects to already-running Vite dev server)
npm run dev:tauri

# Production build
npm run tauri build

# Frontend build only
npm run build

# Rust checks
cd src-tauri && cargo test
cd src-tauri && cargo clippy

# i18n (Lingui)
npm run extract   # extract strings
npm run compile   # compile catalogs

# Versioning (generates changelog, bumps version)
npm run release:patch   # or :minor / :major
```

Version number must stay in sync across three files: `package.json`, `src-tauri/Cargo.toml`, `src-tauri/tauri.conf.json`.

Build artifacts: `src-tauri/target/release/bundle/{msi,dmg,deb,appimage}/`

## Architecture

This is a **Tauri 2.x** desktop app. The frontend communicates with the Rust backend exclusively via `invoke()` calls — there is no REST API between them.

### Frontend (`src/`)

React 18 + Vite + TailwindCSS 4 + shadcn/ui (Radix primitives).

- `components/features/` — one subdirectory per page/feature (`AccountManager/`, `Gateway/`, `KiroConfig/`, `Login/`, `Settings/`, etc.), each with an `index.jsx` entry
- `components/shared/` + `components/layout/` — reusable UI and layout
- `contexts/` — global React contexts: `AccountContext`, `AppSettingsContext`, `DialogContext`, `PrivacyContext`, theme
- `hooks/` — business hooks: `useAutoRefresh`, `useAutoSwitch`, `useModelLock`
- `api/` — thin wrappers around `invoke()` calls, organized by domain (`sessionApi.ts`, `groupTag.ts`)
- `utils/` — pure utility functions
- `routes.tsx` — all route definitions; new pages must also be registered in the sidebar nav

Styles live in `src/index.css`. Dark mode contrast must be verified for every UI change. Select/MultiSelect components need explicit hover and selected-state styles.

### Backend (`src-tauri/src/`)

Rust 2021 edition. All Tauri commands are registered in `main.rs` via `invoke_handler!`.

- `state.rs` — `AppState` (holds `AccountStore`, `GroupTagStore`, `AuthState`, gateway handle, pending login)
- `commands/` — one file per command group (`account_cmd`, `auth_cmd`, `gateway_cmd`, `group_tag_cmd`, `kiro_cli_cmd`, `kiro_settings_cmd`, `machine_guid`, `mcp_cmd`, `powers_cmd`, `steering_cmd`, `skills_cmd`, `hooks_cmd`, `custom_agents_cmd`, `session_manager`, `proxy_cmd`, `update_cmd`)
- `auth/` — OAuth flows; `auth_social.rs` handles Google/GitHub; `providers/` contains IdC (BuilderId/Enterprise) OIDC logic
- `clients/` — HTTP clients for AWS SSO, Kiro Auth, Kiro Portal
- `core/` — `AccountStore`, `GroupTagStore`, deep-link handler, auto-switch logic, `protocol_registry`
- `kiro/` — Kiro IDE integration: `ide` (config file read/write, account switching), `process` (start/stop/detect), `settings` (proxy, model, MCP, etc.), `cli` (SQLite import via `rusqlite`)
- `gateway/` — Axum-based reverse proxy server; handles Anthropic and OpenAI protocol translation, SSE streaming, API key auth, IP allowlist
- `services/session_storage.rs` — session/workspace persistence
- `utils/` — browser detection, command output helpers

### Data locations

- App data: `~/.kiro-account-manager/`
- Kiro IDE config: `~/.kiro/`

## Key Constraints

- Kiro IDE must be **closed** before triggering account switch or machine ID reset; proxy changes require IDE restart.
- The gateway auto-starts on app launch if previously enabled (`gateway::auto_start_if_enabled`).
- The main window is intentionally hidden until the frontend signals ready — avoids startup white flash.
- On platforms with a system tray, closing the window hides it rather than quitting; exit via tray menu.
- Single-instance enforced via `tauri-plugin-single-instance`; deep-link OAuth callbacks are forwarded to the running instance.
- Only Simplified Chinese UI is supported; English and Russian translations have been removed.

## Commit Style

Conventional Commits with Chinese descriptions: `feat: 添加 kiro cli 导入提示`. Minimum regression check before PR: `npm run build && cargo test`.
