# studio/AGENTS.md

Unsloth Studio is the local web UI for chat, training and export. It has three parts:

- `backend/`: FastAPI.
- `frontend/`: React 19, Vite, TanStack Router, Tailwind 4.
- `src-tauri/`: the desktop shell.

Everything here is AGPL-3.0. New files start with:
`# SPDX-License-Identifier: AGPL-3.0-only` and
`# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0`
(use `//` instead of `#` in TypeScript). The root `AGENTS.md` still applies to Python
formatting, the Python floor and installer parity.

## Layout

- **Routes.** `backend/main.py` builds the app and mounts each router from `backend/routes/<area>.py` at `/api/<area>`.
  The logic lives in `backend/core/`, and helpers live in `backend/utils/`.
- **Backend imports.** These are rooted at `studio/backend`, as in `from core.inference import ...` and `from utils.utils import ...`.
  `run.py` and `backend/tests/conftest.py` put `backend/` on `sys.path`.
- **Two test suites.** `backend/tests/` holds the backend unit tests. The repo-root `tests/studio/` is a different suite:
  it guards the installers, the CI workflows and studiobench.
- **Frontend layout.** Code is in `frontend/src/features/<feature>/`, and translations are in `frontend/src/i18n/`.

## Commands (verified on macOS arm64)

```bash
# backend: on top of the root dev install
uv pip install -r studio/backend/requirements/studio.txt pytest-asyncio pytest-timeout httpx
cd studio/backend && python -m pytest -q -n 4 --dist loadgroup tests/test_auth*.py
UNSLOTH_STUDIO_HOME="$PWD/.studio" python studio/backend/run.py --port 8888 --api-only  # from repo root; GET /api/health

# frontend: Node 22 (`nvm use 22`)
cd studio/frontend && npm ci --ignore-scripts --no-fund --no-audit
npm run dev                 # Vite; proxies /api and /v1 to 127.0.0.1:8888
npm test                    # node:test, ~14k tests, ~80 s
npm run typecheck
npm run i18n:check:strict   # CI gate
npx biome check --write src/path/changed.tsx
```

## Gotchas

- **Node 22 is required.** `package.json` `engines` admits Node 20.19, but `npm test` and the
  `i18n:*` / `catalog:*` scripts use `--experimental-strip-types`, so Node 20 fails with
  `node: bad option`. CI pins Node 22.
- **Scope Biome to your files.** Running Biome over the whole tree already reports errors on `main`, and CI runs it
  as non-blocking. `npm run biome:fix` would therefore rewrite unrelated files.
- **Set `UNSLOTH_STUDIO_HOME` when you start the backend.** Without it, the backend reads and writes `~/.unsloth/studio`,
  which is the user's real install (accounts, models, settings). `/.studio/` is gitignored.
- **Pair `--api-only` with `npm run dev`.** Without `--api-only`, `run.py` exits unless `frontend/dist/` exists.
  `npm run build` creates it.
- **New UI strings.** Add the key to `src/i18n/locales/en.ts` *and* to all 13 other locale overlays in the
  same change, because the strict check fails on missing keys. Rules are in `src/i18n/README.md`.
- **Vendored code.** `backend/vendor/` is byte-identical upstream code. Reach it through `sys.path`, never as
  `studio.backend.vendor.*`. See `backend/vendor/README.md`.
- **Generated files.** Regenerate `src/features/chat/model-catalog-snapshot.ts` with `npm run catalog:refresh`.
  Change `package-lock.json` only through npm. The pre-commit hook `sync-allow-scripts-pins` re-pins
  `allowScripts` in `package.json`.
