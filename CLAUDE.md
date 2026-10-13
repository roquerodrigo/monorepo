# CLAUDE.md

pnpm workspace (see `pnpm-workspace.yaml`) holding Railyard (Go + Wails desktop app with a React frontend),
the website, the Foundry map tools and shared `@subway-builder-modded/*` packages. Install once from the
repo root; sub-packages never need their own install.

## Gotchas

- `.gitignore` lists `CLAUDE.md` and `AGENTS.md` (upstream ignores agent files). This file is tracked
  anyway; a new agent file elsewhere needs `git add -f`.
- README's `pnpm run ci` does not exist at the root. Root scripts are `check`, `test`, `lint`, `format`
  and their `:packages` / `:website` / `:railyard` variants (`package.json`). `ci` exists only inside
  `website/` and `railyard/frontend/`, and it auto-fixes (`lint:fix` + `format`); use `check` to verify.
- README says production deploys from a `website` branch. It does not: `website-deploy.yml` deploys
  `main` on push and also hourly (cron `20 * * * *`). `website-dev` deploys to a separate repo
  (`dev.subwaybuildermodded.com`).
- The two frontends use different toolchains. `website/` lints and formats with vite-plus
  (`vp lint` / `vp fmt`); `railyard/frontend/` uses ESLint + Prettier. Run each app's own scripts, not
  the other's tool.
- Shared packages under `packages/*` have no build step: `exports['.']` points at `./src/index.ts` and
  consumers compile the source. Packages CI fails unless each package is named `@subway-builder-modded/*`
  and has `exports['.']` whose `types`/`default` targets exist. Changes there run the website and
  railyard frontend CI as consumers.
- `.githooks/pre-push` runs `website/scripts/quality.ps1` (or `quality.sh`) when `website/` changed, but
  neither script exists, so with `core.hooksPath .githooks` set, any push touching `website/` fails.
  Railyard changes run `railyard/scripts/pre-push-check.*`, which rewrites files (`gofmt -w`,
  `format`, `lint:fix`) before checking.

## Railyard (`railyard/`)

- `main.go` embeds `frontend/dist` (`//go:embed all:frontend/dist`), which is gitignored. Plain
  `go test ./...` fails on a fresh clone until that directory exists; `scripts/backend-quality.sh`
  creates a `.embed-placeholder` for this reason. Run backend checks via that script.
- Go coverage gate: `scripts/check-go-coverage.sh`, minimum 60% total (override with `GO_COVER_MIN`),
  excluding `internal/testutil`.
- `frontend/wailsjs/` is Wails-generated bindings but is committed. Changing exported methods on bound
  Go structs requires regenerating them (`wails dev`/`wails build` does) and committing the result.
- `app_appimage.go` is `//go:build linux`; `go_appimage_stub.go` (`!linux`) must keep matching
  signatures or non-Linux builds break.
- Wails runs `frontend:install` as `pnpm --dir ../.. install --frozen-lockfile` (workspace root), so
  frontend dependency changes must land in the root `pnpm-lock.yaml`.
- Versioning: release CI writes the tag into `internal/constants/version.txt` and `wails.json`
  (`scripts/update_wails_json.js`, reads `VERSION`). The committed values are not the source of truth.
  On Windows, the pre-build hook `scripts/generate-version.ps1` derives the NSIS version from
  `wails.json`.
- Releases: pushing a `v*` tag builds macOS/Linux (Flatpak)/Windows. A tag containing `+rc` builds but
  skips creating the GitHub release.

## Website (`website/`)

- `pnpm run fetch` clones `Subway-Builder-Modded/registry` and fills the gitignored
  `public/registry-cache/` and related folders; it needs `SBM_GITHUB_TOKEN` (`.env`/`.env.local`, see
  `.env.template`). Both deploy workflows run it before `build`.
- Tests load shared jsdom mocks from the repo-level `testing/jsdom.ts`.

## Foundry schemas (`foundry/schemas/`)

- Published to GitHub Packages by pushing tag `special-demand-schemas-v<version>`; the tag must equal
  `foundry/schemas/package.json` `version`. Steps are in `foundry/schemas/RELEASING.md`.
- CI checks the packed tarball: the three `*.schema.json` files and `dist/` must be included, and the
  example fixtures (`special_demand_types.json`, `special_demand_points_*.json`) must not be.
