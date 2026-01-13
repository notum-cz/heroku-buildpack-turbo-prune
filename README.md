# Heroku Buildpack: Turbo Prune for Monorepo

This buildpack runs `turbo prune --docker` **before** the Heroku Node.js buildpack installs dependencies.

Result: **faster installs** and a **smaller slug** when deploying a Turborepo monorepo to Heroku.

- Intended for deploying **multiple separate Heroku apps from the same repo**, where each app prunes to a different workspace scope.
- Intended for the **[strapi-next-monorepo-starter](https://github.com/notum-cz/strapi-next-monorepo-starter)** monorepo (Strapi + Next.js) deployed to Heroku, but should work in other Turborepo monorepos as well.

## Installation on Heroku

Add buildpacks **in this order**:

1. `https://github.com/notum-cz/heroku-buildpack-turbo-prune`
2. `heroku/nodejs`
---
3. (optional) `https://github.com/notum-cz/heroku-buildpack-next-standalone-slim` (additional buildpack when deploying a Next.js app with `standalone` output)

## How it works

[Heroku’s Node.js buildpack](https://github.com/heroku/heroku-buildpack-nodejs) installs dependencies first and only then runs `build` (or [`heroku-postbuild`](https://devcenter.heroku.com/articles/nodejs-classic-buildpack-builds#add-steps-before-or-after-heroku-specific-build-steps)). In a monorepo that often installs far more dependencies than needed.

This buildpack changes the order:

1. **turbo-prune buildpack** downloads a standalone `turbo` binary and runs `turbo prune --docker` for one workspace scope
2. **heroku/nodejs** installs dependencies only for the pruned output
3. **heroku/nodejs** runs `build` in the pruned output (only for selected workspace)

## Requirements

- A Turborepo repo with `turbo.json` at the repo root
- Heroku stack running Linux x64 (Heroku-22/24)
- Your apps live under `apps/<app>` (e.g. `apps/ui`, `apps/strapi`)

## Configuration

### Required config vars

These are set **per Heroku app/dyno** as Heroku config vars:

- `WORKSPACE` – the scope passed to `turbo prune` - e.g. `@repo/ui`, `@repo/strapi`
- `APP` – the app folder name under `apps/` - e.g. `ui`, `strapi`

### Optional config vars

- `TURBO_VERSION` – turbo version to download (default: `2.7.3`). Should match the version in your repo's package.json

## Heroku artifacts copied to repo root

Heroku expects certain files in the **build root**. Because `turbo prune` replaces the repo contents, this buildpack copies these files from `apps/$APP/` to the build root after pruning:

- `apps/$APP/Procfile` → `./Procfile` (optional)
- `apps/$APP/.slugignore` → `./.slugignore` (optional)
- `apps/$APP/app.json` → `./app.json` (optional)

This behavior is similar to [heroku-buildpack-multi-procfile](https://github.com/heroku/heroku-buildpack-multi-procfile/tree/master) but integrated into this buildpack.

## Special handling: `@repo/strapi-types` generated types

For the starter, the UI build depends on generated `.d.ts` files.
Before pruning, the buildpack materializes `packages/strapi-types/generated/*.d.ts` from `apps/strapi/types/generated/*.d.ts` so the types survive pruning. This is specific to the [strapi-next-monorepo-starter](https://github.com/notum-cz/strapi-next-monorepo-starter) and will be skipped in other repos.
