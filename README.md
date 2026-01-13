# Heroku Buildpack: Turbo Prune for Turborepo Monorepos

This buildpack runs `turbo prune --docker` **before** the Heroku Node.js buildpack installs dependencies.

Result: **faster installs** and a **smaller slug** when deploying a Turborepo monorepo to Heroku.

- It is supposed to be used with the **[strapi-next-monorepo-starter](https://github.com/notum-cz/strapi-next-monorepo-starter) repository**.
- It is intended for deploying **two separate Heroku apps from the same repo** (UI + Strapi), where each app prunes to a different scope.

## Why this buildpack exists

Heroku’s Node.js buildpack runs `yarn install` first and only then executes `heroku-postbuild`. In a monorepo, that means you normally install dependencies for the entire repo, even if you deploy only one app.

This buildpack changes the order:

1. **turbo-prune buildpack** prunes the repo to a single scope
2. **heroku/nodejs** installs dependencies only for the pruned output
3. Your usual build scripts run on the pruned repo

## Requirements

- A Turborepo repo with a turbo.json at the repo root
- `yarn` workspaces (works well with Yarn v1 as in the starter)
- Heroku stack running Linux x64 (standard Heroku stacks)
- Your app(s) deployed as **two Heroku apps** from the same repo (recommended)

## Configuration

### Required config var

- `TURBO_SCOPE` – the scope passed to `turbo prune`, e.g.:
  - `@repo/ui`
  - `@repo/strapi`

### Optional config var

- `TURBO_VERSION` – pass specific version or pin to `2.7.3`

## Special handling: `@repo/strapi-types` generated types

For the `strapi-next-monorepo-starter`, the UI build depends on generated `.d.ts` files. The buildpack will materialize:
`packages/strapi-types/generated/*.d.ts` before pruning, so the types survive pruning.

No extra configuration is needed.

## Installation on Heroku

You must add **two buildpacks**, in this order:

1. `https://github.com/notum-cz/heroku-buildpack-strapi-next-monorepo-prune.git`
2. `heroku/nodejs`
