# npm cheat sheet: cache, lockfiles, npm vs pnpm

Full guides: [npm cache clean](https://bigdomainhost.com/guide-npm-cache-clean.html) · [npm vs pnpm](https://bigdomainhost.com/guide-npm-vs-pnpm.html)

## Three layers, three resets

| Layer | Where | Reset with | Fixes |
|---|---|---|---|
| Cache | `~/.npm/_cacache` (Linux/macOS), `%LocalAppData%\npm-cache` (Windows) | `npm cache verify`, `npm cache clean --force` | corrupt or bad downloaded tarballs (`EINTEGRITY`) |
| Installed tree | `node_modules/` | delete it, or `npm ci` | half-finished installs, wrong native builds |
| Plan | `package-lock.json` | delete and reinstall (last resort) | resolution you actually want changed |

Clearing the cache never touches `node_modules` or the lockfile.

## Cache

```bash
npm config get cache          # where the cache lives
npm cache verify              # check integrity, drop bad entries, keep good ones
npm cache clean --force       # delete the whole cache (--force is required)
npm install --prefer-online   # revalidate cached metadata against the registry this once
npm install --cache /tmp/empty-cache   # test with a throwaway cache, leave the real one alone
```

Try `verify` first. The cache checks itself on every read and rarely needs wiping.

## Clean reinstall (most install errors)

```bash
rm -rf node_modules
npm ci                        # installs exactly what package-lock.json says; fails if it disagrees with package.json
```

PowerShell: `Remove-Item -Recurse -Force node_modules; npm ci`

## Lockfile rules

- Commit `package-lock.json`. Use `npm ci` in CI, Docker and deploys; `npm install` on your machine when changing dependencies.
- `npm ci` errors when the lockfile and `package.json` do not match: run `npm install` locally, commit the updated lockfile.
- Regenerate the lockfile without touching `node_modules`: `npm install --package-lock-only`.
- One package manager per project. npm ignores `pnpm-lock.yaml` and `yarn.lock`; pnpm ignores `package-lock.json`.

## Find out what is installed and why

```bash
npm ls express                # where it sits in the tree
npm explain express           # why it is installed (alias: npm why)
npm outdated                  # current vs wanted vs latest
npm dedupe                    # flatten duplicate versions
npm audit                     # known vulnerabilities
npm audit fix                 # apply semver-compatible fixes only
npm doctor                    # checks registry access, cache, permissions, node/npm versions
```

## npm vs pnpm

| Task | npm | pnpm |
|---|---|---|
| Install from package.json | `npm install` | `pnpm install` |
| Add a dependency | `npm install express` | `pnpm add express` |
| Add a dev dependency | `npm install -D vitest` | `pnpm add -D vitest` |
| Remove | `npm uninstall express` | `pnpm remove express` |
| Run a script | `npm run build` | `pnpm build` or `pnpm run build` |
| Run a package without installing | `npx create-vue my-app` | `pnpm dlx create-vue my-app` |
| Run a project binary | `npx eslint .` | `pnpm exec eslint .` |
| Clean CI install | `npm ci` | `pnpm install --frozen-lockfile` |
| Global tool | `npm install -g pm2` | `pnpm add -g pm2` |
| Why is it here | `npm explain pkg` | `pnpm why pkg` |
| Lockfile | `package-lock.json` | `pnpm-lock.yaml` |

pnpm cache (the content-addressable store):

```bash
pnpm store path               # where it is
pnpm store prune              # remove packages no project references
```

Switch a project from npm to pnpm:

```bash
pnpm import                   # builds pnpm-lock.yaml from package-lock.json
rm -rf node_modules
pnpm install
# test the build, then delete package-lock.json and update scripts/CI/Dockerfile
```

pnpm only links declared dependencies at the top of `node_modules`, so an import that "always worked" under npm can fail with `Cannot find module`. Fix it by adding the package: `pnpm add <pkg>`.

## Yarn, for completeness

```bash
yarn cache clean              # Yarn 1 and Yarn Berry
yarn install --frozen-lockfile   # Yarn 1 CI install
yarn install --immutable         # Yarn 2+ CI install
```
