# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Layout

The git root contains a single Vue 3 + Vite app in `StockyApp/`. All commands below run from `StockyApp/`. There is also a stray `package-lock.json` at the repo root (no root `package.json`).

The app is currently the unmodified `create-vue` starter template (`HelloWorld`, `TheWelcome`, `WelcomeItem`, and icon components); no stock-specific features exist yet.

## Commands

```sh
npm install
npm run dev       # Vite dev server with HMR
npm run build     # production build to dist/
npm run preview   # serve the production build
```

There is no test runner or linter configured. Node `^22.18.0 || >=24.12.0` is required (`engines` in `package.json`).

## Architecture

- Plain JavaScript (no TypeScript), Vue 3 SFCs using `<script setup>`; no router or state library.
- Entry: `index.html` → `src/main.js` (imports `assets/main.css`, mounts `App.vue` on `#app`).
- `@` is aliased to `src/` in both `vite.config.js` and `jsconfig.json`; keep the two in sync if it changes.
- `vite-plugin-vue-devtools` is enabled in the Vite config.
