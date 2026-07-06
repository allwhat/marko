# PackageMaze CircleCI Rehearsal Notes

This notes file records the PackageMaze setup rehearsal for `allwhat/marko`.
It is intended as input for improving PackageMaze's public setup guidance and
MCP behavior, not as Marko product feedback.

## Repository And Branches

- Source checkout: `/Users/kkoz/projects/temp/marko`
- Published repository: `https://github.com/allwhat/marko`
- Baseline pushed to `main`: upstream Marko commit `98ea020a54`
- CircleCI UI generated branch: `circleci-project-setup`
- CircleCI UI generated commit: `119894cd73d2fdf51bdad81c58297268eff294ff`
- PackageMaze rehearsal PR: `https://github.com/allwhat/marko/pull/1`
- PackageMaze rehearsal branch: `codex/circleci-packagemaze`

## PackageMaze Surfaces Used

I used PackageMaze as an external customer agent after the user clarified that
local PackageMaze source code should not be used.

MCP tools called:

- `package_maze.get_setup_context`
- `package_maze.plan_repository_setup`
- `package_maze.review_repository_setup`
- `package_maze.get_package_activity`

MCP resources read:

- `packagemaze://mcp/setup/repository-setup`
- `packagemaze://mcp/setup/credentials-and-ci`

Public repositories and docs used:

- `packagemaze/maze-cli` README
- `packagemaze/setup-maze` README
- `packagemaze/maze-cli` release metadata for `v0.0.2`
- CircleCI OIDC docs for `circleci run oidc get --claims`
- CircleCI `cimg/node` image docs

## What PackageMaze Reported

`get_setup_context` exposed several npm Feeds, so Feed selection was ambiguous.
The user confirmed `packagemaze/dogfood-npm`. That Feed has upstream npmjs
proxying enabled and a Feed Base URL of:

```text
https://pkg.packagemaze.com/packagemaze/dogfood-npm/
```

Repository discovery found:

- npm workspaces at the repository root
- `package-lock.json` lockfile version 3
- no committed `.npmrc`
- all sampled lockfile `resolved` hosts were `registry.npmjs.org`
- existing GitHub Actions install and release workflows
- no existing CircleCI config before CircleCI UI setup
- no Python, pnpm, Yarn, Bun, or Dockerfile package-client surface

The useful PackageMaze recommendation was:

```ini
registry=https://pkg.packagemaze.com/packagemaze/dogfood-npm/
replace-registry-host=npmjs
```

That advice fits this repository because npm can rewrite `registry.npmjs.org`
lockfile hosts through the configured PackageMaze registry when the Feed links
the default npmjs upstream.

## What I Tried First

The first PR version overreached. I replaced CircleCI setup with a custom
PackageMaze-oriented `.circleci/config.yml` that:

- installed the PackageMaze `maze` CLI
- requested a CircleCI OIDC token
- exchanged it for PackageMaze install and publish Tokens
- wrote a trusted temporary npm config
- created custom build, test, and publish jobs

This worked as a possible hand-written CircleCI integration, but it made
PackageMaze appear responsible for CircleCI project scaffolding and release
workflow design. That is the wrong ownership boundary.

## CircleCI UI Setup Result

The user then set up the project in the CircleCI UI. CircleCI created commit
`119894cd73d2fdf51bdad81c58297268eff294ff`, which only added:

```text
.circleci/config.yml
```

The generated config:

- used `circleci/node@5`
- used `node/install-packages` with `pkg-manager: npm`
- ran `npm test --passWithNoTests`
- ran `npm run build`
- added generic artifact copying
- included a placeholder deploy job

It did not add PackageMaze npm registry configuration, PackageMaze auth, or
publish registry configuration. That separation is healthy: CircleCI owns CI
project setup; PackageMaze owns package-client configuration.

## What I Changed After That

The PR was updated to keep CircleCI's generated workflow shape and add only
PackageMaze package-client pieces:

- committed root `.npmrc` with the dogfood Feed Base URL and npmjs host
  replacement
- added `publishConfig.registry` for publishable npm workspace packages
- inserted a small `configure-packagemaze-npm` CircleCI command before
  CircleCI's existing `node/install-packages` steps

The CircleCI command installs `maze`, gets a CircleCI OIDC token with audience
`https://api.packagemaze.com`, exchanges it for a PackageMaze install Token,
and writes a temporary `NPM_CONFIG_USERCONFIG` containing the PackageMaze token.
This keeps credentials out of committed files and leaves CircleCI's generated
job structure intact.

## Second PackageMaze Review Result

After narrowing the PR to CircleCI's generated workflow plus a PackageMaze npm
auth command, I called `package_maze.review_repository_setup` again. The review
still returned GitHub Actions-oriented diagnostics. It did not recognize
CircleCI's `node/install-packages` orb command as the supported npm package
install surface, even though it was included in the submitted workflow commands.

The review also flagged the rehearsal notes file itself as an overbroad
PackageMaze change because the file mentions PackageMaze but is not a package
client surface. That warning is technically understandable from a static
keyword perspective, but it is not useful for an agent writing setup notes.

The useful lesson from the second review is that PackageMaze should understand
CircleCI-native package install abstractions, especially:

- `circleci/node` orb `node/install-packages`
- a PackageMaze auth/setup command immediately before that orb command
- CircleCI `BASH_ENV`-based propagation of `NPM_CONFIG_USERCONFIG`

After the PR was picked up by CircleCI, PackageMaze showed the CircleCI OIDC
exchange but no package installs through the Feed. That means the auth exchange
step ran, but the package-client install step did not produce observable
PackageMaze package traffic. The likely causes are either the CircleCI node orb
install abstraction using cache in a way that avoided registry fetches, or the
orb not honoring the expected npm config environment. The PR was then adjusted
to use an explicit `npm ci` step with a fresh per-job npm cache directory after
the PackageMaze config step so the package client should make observable
registry requests.

This should become PackageMaze guidance: when the user wants to verify package
traffic, cache-aware CI abstractions can hide installs. PackageMaze should
either recognize and explain that or recommend an explicit package-client
install command for the verification run.

The explicit `npm ci` run printed:

```text
https://pkg.packagemaze.com/packagemaze/dogfood-npm/
npmjs
added 842 packages, and audited 847 packages in 59s
```

That confirmed the runtime npm config was present, but not yet that the tarball
fetches were actually routed through PackageMaze. The next diagnostic added to
the CircleCI install step was `--loglevel=http`.

The user then supplied the CircleCI HTTP log from that run at
`/Users/kkoz/Downloads/maze_log.log`. After stripping ANSI color codes, the log
showed:

```text
PackageMaze feed URL lines: 840
PackageMaze GET 200 tarball fetches: 838
PackageMaze POST 200 advisory requests: 1
cache misses: 838
registry.npmjs.org lines: 0
```

Representative lines included successful `GET 200` tarball fetches for
`yocto-queue`, `typescript`, and `@babel/core` from:

```text
https://pkg.packagemaze.com/packagemaze/dogfood-npm/
```

The log resolves the configuration question: npm did install through
PackageMaze, including lockfile-host replacement for upstream npm packages.
There were no direct `registry.npmjs.org` fetches in this install log.

After that evidence, I rechecked PackageMaze package activity through MCP:

- `repository=allwhat/marko`, `query=yocto-queue`: zero Package Usage rows and
  zero Package Resolution rows
- unfiltered `query=yocto-queue`: zero rows
- unfiltered `query=typescript`: zero rows
- `source_type=proxied_upstream`: zero rows

The remaining issue is therefore not the customer repository configuration. It
is PackageMaze visibility: recording, read-model projection, UI filtering, or
CI/repository attribution for proxied upstream npm requests served through the
Package Client Domain.

## PackageMaze Friction And Gaps

CircleCI-specific guidance is too thin in the MCP setup flow. The public
`maze-cli` README has a basic CircleCI path, but the MCP planning and review
language still heavily assumes GitHub Actions and `packagemaze/setup-maze`.

`plan_repository_setup` returned actionable npm registry lines, but it also
suggested GitHub Actions access even for a submitted `.circleci/config.yml`.
For CircleCI, PackageMaze should guide the package-client auth step around an
existing CircleCI install command rather than ask for `setup-maze`.

`review_repository_setup` produced useful warnings for the CircleCI-generated
branch: `.npmrc` was missing and lockfile traffic still looked like direct
npmjs access. Those are the right PackageMaze-owned concerns.

`review_repository_setup` also produced noisy or incorrect warnings:

- it treated CircleCI evidence as "GitHub Actions evidence" and asked for
  `packagemaze/setup-maze@v0.0.3`
- it treated a CircleCI Docker executor as a Docker/BuildKit package install
  surface
- it treated a submitted repository URL as an outside package source when I
  included it in `current_sources`, which suggests the tool contract needs
  clearer examples for `current_sources`

The MCP planner suggested per-workspace `.npmrc` files in addition to the root
`.npmrc`. For this npm workspace with one root lockfile, the root `.npmrc` is
the right default. Extra workspace `.npmrc` files would be redundant.

The PackageMaze CircleCI path currently relies on manually installing the
`maze` CLI from GitHub release assets. A first-class CircleCI orb, reusable
command, or clearer copy-paste snippet would make the customer path less
hand-rolled.

The public troubleshooting surface has a visibility gap after successful OIDC
exchange. In this run, PackageMaze showed the CircleCI OIDC exchange, npm logs
showed hundreds of successful Package Client Domain requests, but PackageMaze
Package Usage and Resolution History remained empty through MCP queries. As an
external user agent, I had no public way to ask PackageMaze to correlate the
OIDC exchange, minted Token, Feed, CI run, repository, and package-client HTTP
requests.

The docs should explicitly distinguish:

- CircleCI project setup: user and CircleCI responsibility
- CircleCI workflow shape: user and CircleCI responsibility
- package-client registry settings: PackageMaze responsibility
- package-client CI auth around install/publish steps: PackageMaze guidance
- package publish workflow design: user responsibility unless already present

For existing CircleCI workflows, PackageMaze should show small patches around
the user's actual package-client commands, for example "add this before
`node/install-packages`" or "add this before `npm ci`", instead of generating a
whole workflow.

## Suggested PackageMaze Product Improvements

- Make CircleCI a first-class setup target in MCP planning and review, not a
  GitHub Actions variant.
- Return a minimal CircleCI snippet for npm installs:
  - install or provide `maze`
  - run `circleci run oidc get --claims '{"aud":"https://api.packagemaze.com"}'`
  - exchange with `maze auth exchange-oidc`
  - write a temporary npm config
  - set `NPM_CONFIG_USERCONFIG` before the install step
- Add a CircleCI-specific review rule that recognizes `node/install-packages`
  as the package-client install step.
- Warn when a CI package install abstraction or restored cache may produce an
  OIDC exchange but no package downloads visible to PackageMaze.
- Record and surface proxied upstream npm tarball downloads served through the
  Package Client Domain, including enough Feed, Token, CI, and repository
  context for a user to understand the install path.
- Add a feed or CI-session "doctor" that can say: OIDC exchange succeeded, Token
  minted, package-client requests observed, package activity rows created or
  missing.
- Let MCP query package-client request evidence by Feed, CI session, run,
  repository, and package name, even if Package Usage or Package Resolution rows
  were not projected.
- Stop classifying CircleCI Docker executors as Docker image builds unless a
  Docker build command or Dockerfile package-client install is present.
- Prefer the nearest project package-client config once for npm workspaces with
  one root lockfile; do not suggest duplicate per-workspace `.npmrc` files by
  default.
- Make `current_sources` examples clearer so agents do not submit repository
  URLs as package sources.
- Be explicit that PackageMaze should not create or own a CircleCI project.
  PackageMaze should verify package-client routing after CircleCI exists.
- Provide separate guidance for install and publish:
  - installs: `.npmrc` plus CI auth before install commands
  - publishes: `publishConfig.registry` or `npm publish --registry`, plus a
    publish Token only where the user already has a publish workflow

## Remaining External Checks

I could parse YAML and package manifests locally, but I could not fully validate
CircleCI behavior because the local environment has no CircleCI CLI or API
token. The real end-to-end checks are:

- whether CircleCI picks up PR updates on `codex/circleci-packagemaze`
- whether the configured CircleCI project can mint OIDC tokens
- whether PackageMaze has matching CircleCI install access for
  `7530b3e9-c7bc-4556-9ac5-0fecb6036085`
- whether `node/install-packages` honors `NPM_CONFIG_USERCONFIG` from
  `BASH_ENV`
- whether PackageMaze records Package Usage and Resolution History for the run
