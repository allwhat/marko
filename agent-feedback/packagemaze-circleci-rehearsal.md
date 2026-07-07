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

The next full CircleCI log showed the `Run tests` step failing with a Node/V8
out-of-memory error after running the Marko Mocha suite for about two and a half
minutes. The failing command was `npm test --passWithNoTests`; npm warned that
`--passWithNoTests` is an unknown npm config, so that flag was not useful here.
Because this rehearsal is about PackageMaze install routing, not Marko test
coverage, I disabled the CircleCI test execution while preserving the
PackageMaze-backed dependency install step.

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

## Internal Source And Cloudflare Investigation

After the user allowed PackageMaze source, Cloudflare, and production storage
inspection, I checked the production CI Session and the relevant read models.

The CircleCI side was healthy:

- latest successful CircleCI pipeline/run:
  `ea51eef9-decb-4584-9fb2-386d62e286d1`
- CircleCI workflow id:
  `27c162b4-ac8d-426e-8561-4203b29d5e5f`
- PackageMaze CI Session:
  `cis_70847ee83a4d37cf32f3f174f17401cc6a60a5fe55cd0dbfec5580338aedd61b`
- PackageMaze production D1 `ci_sessions` had the matching repository,
  run id, workflow ref, branch ref, and two successful OIDC exchanges.
- The two minted CI Tokens were linked through `token_ci_contexts` to the same
  CI Session and had `last_used_at` timestamps during the CircleCI jobs:
  - `tok_ee80713e6e0d4fd3b8d95c784b93765e` last used at
    `2026-07-06T20:58:26Z`
  - `tok_244097908def44eeac4a481e949168a5` last used at
    `2026-07-06T20:59:28Z`

Production aggregate usage also proved PackageMaze observed package-client
traffic:

- `usage_event_dedupe` contained 1,676 `artifact_download_usage_v1` events
  between `2026-07-06T20:57:40Z` and `2026-07-06T20:59:28Z`, matching two npm
  install jobs at roughly 838 tarball downloads each.
- `usage_daily` and `usage_period_daily` had npm `artifact_blob_download`
  aggregate rows for the dogfood Feed, source `cached`, status `completed`,
  and latest event time `2026-07-06T20:59:28Z`.
- The R2 usage ledger had per-event package details. Example records included:
  - `wrappy@1.0.2`, token
    `tok_ee80713e6e0d4fd3b8d95c784b93765e`, serving source `upstream`,
    `packageVersionId: null`
  - `webidl-conversions@8.0.1`, token
    `tok_244097908def44eeac4a481e949168a5`, serving source `upstream`,
    `packageVersionId: null`

The empty Package Usage and Resolution History is therefore a PackageMaze
read-model/product semantics issue, not a Marko or CircleCI setup issue.

I also checked the duplicate tarball lines visible in the CircleCI npm HTTP
logs. The step output for CircleCI job `17` contained 838 npm tarball `GET`
lines through the PackageMaze Feed and 778 unique tarball URLs. The duplicate
requests line up with duplicated package locations in `package-lock.json`.
For example, npm fetched `prettier-2.8.8.tgz` twice:

```text
npm http fetch GET 200 https://pkg.packagemaze.com/packagemaze/dogfood-npm/prettier/-/prettier-2.8.8.tgz 50834ms (cache miss)
npm http fetch GET 200 https://pkg.packagemaze.com/packagemaze/dogfood-npm/prettier/-/prettier-2.8.8.tgz 50919ms (cache miss)
```

The lockfile has two distinct package nodes that resolve to that tarball:

```text
node_modules/@changesets/apply-release-plan/node_modules/prettier
node_modules/@changesets/write/node_modules/prettier
```

This is not evidence that CircleCI duplicated the step or that PackageMaze
configured npm incorrectly. It is npm installing two package locations from a
fresh per-job cache, with concurrent tarball fetches completing at nearly the
same time. The PackageMaze-specific expectation is that duplicate artifact
requests are harmless: they should be recorded in the CI Session, should warm or
hit the PackageMaze cache on subsequent runs, and ideally should be coalesced
while an identical cold upstream fetch is already in flight.

I also investigated why the PackageMaze CI Session for CircleCI pipeline
`a701f999-d8bf-4e77-8008-151ae3503554` showed four OIDC exchanges when the
workflow shape appears to have only `test-node` and `build-node`. CircleCI had
two workflow executions under that same pipeline id:

```text
d9731bb7-5390-4dc0-b7b1-809f3672d1e1  build-and-test  2026-07-06T21:28:56Z
051f058f-d559-4980-9aa6-7880e59f23c2  build-and-test  2026-07-07T01:33:08Z
```

Each workflow execution had one `test-node` job and one `build-node` job, so
PackageMaze recorded two exchanges for the original workflow execution and two
for the rerun. PackageMaze currently keys CircleCI CI Sessions by repository and
CircleCI pipeline id, so workflow reruns under the same pipeline are grouped
into one Session. That is explainable from the current implementation, but it is
confusing against the documented "one CI provider run attempt" Session language
and against the user's expectation that the selected workflow would show two
exchanges.

The Session UI also rendered each exchange as `install Package not requested`.
That text means `requested_package_name` on the OIDC exchange request was null:
the job requested a Feed-wide install Token, not a package-scoped Token. It does
not mean npm failed to request or download packages from the Feed. This wording
is misleading in exactly the setup-debugging flow where users are trying to
confirm package activity.

I filed PackageMaze follow-up issue
`https://github.com/packagemaze/packagemaze/issues/1479` to clarify CircleCI
workflow execution grouping and replace the `Package not requested` copy with
language such as "Feed-wide token" or "No package scope".

Follow-up checks on CircleCI OIDC showed:

- `oidc.circleci.com/ssh-rerun` was false for all four exchanges. It is useful
  for rejecting CircleCI SSH debug reruns, not for identifying a normal workflow
  rerun.
- `oidc.circleci.com/workflow-id` is the useful rerun discriminator in this
  case. The original workflow execution and rerun had different workflow ids
  under the same pipeline id.
- CircleCI OIDC includes `job-id`, and the CircleCI API maps those job ids to
  `test-node` and `build-node`. PackageMaze does not currently enrich Session
  rows from the CircleCI API.
- CircleCI OIDC does not include the workflow name, and CircleCI built-in
  environment variables expose `CIRCLE_WORKFLOW_ID` but not a workflow-name
  variable. The workflow name `build-and-test` is available through the
  CircleCI API when querying by workflow id.
- The PackageMaze CI token exchange route already accepts
  `setup_invocation_id` and bounded `client` context, but the dogfood CircleCI
  snippet did not send either. A PackageMaze CircleCI orb or setup command could
  use that path to capture job/setup context explicitly.

The UI direction that now seems most useful is an exchange-centric Session view:
group by workflow execution, then job or setup invocation, then show exchange
attempt, minted Token, and observed package activity. Separate Token and OIDC
exchange tables are accurate as backend concepts, but for setup debugging they
make the user mentally join rows that belong together.

Relevant source paths:

- `runtime-worker/src/npm/package-routes.ts`
  - cold upstream npm tarball serving calls `enqueueArtifactUsageMetering`
    through `meteredNpmColdUpstreamArtifactBody`
  - `npmColdUpstreamUsageDescriptor` records package name, version, size, and
    `source: "cached"`, but sets `packageVersionId: null`
- `runtime-worker/src/product-state/download-usage.ts`
  - `enqueueArtifactUsageMetering` creates aggregate metering only
  - Package Usage History rollups are only written when the queued message has
    `packageUsageActivity`, which requires a retained Package Version id
  - the existing test explicitly covers "artifact metering without Package
    Usage History activity"
- `runtime-worker/src/product-state/package-resolution-usage.ts`
  - Package Resolution History rows are written only when a package version id
    or policy-withheld package version id exists
  - cold upstream metadata usage has no package version id, so no Package
    Resolution History row is created
- `runtime-worker/src/product-state/repository.ts`
  - CI Session `observedPackageMazeTraffic` is computed from
    `package_usage_activity_rollups` plus `package_resolution_activities`
    only
  - aggregate `usage_daily` traffic and R2 usage-ledger evidence do not count
- `runtime-worker/src/hosted-mcp/tools/ci-session-report.ts` and
  `frontend/src/components/OrganizationSessionsWorkspace.tsx`
  - the MCP and UI then report "minted tokens but did not observe package
    activity" even when PackageMaze did observe package-client tarball traffic

Recommended PackageMaze follow-up:

- Change the CI Session report terminology so "observed PackageMaze traffic"
  means package-client traffic that reached PackageMaze, not only known
  Package Version history rows.
- Add a separate signal such as "observed package-client requests" based on
  aggregate usage or a new package-client request read model.
- Surface cold upstream package names and versions from the usage ledger or a
  durable D1 projection, even when `packageVersionId` is null.
- Decide whether successful cold upstream tarball installs should immediately
  create retained Package Version records, or whether Package Usage History
  should remain known-retained-artifact-only and the CI Session view should get
  a distinct "observed upstream package-client traffic" table.
- Update the UI and MCP copy so users do not debug CircleCI/npm config when the
  actual state is "requests were observed, but no known Package Version history
  rows were projected."

## Artifact Fill Throughput Investigation

After the user pointed out that the install looked catastrophically slow, I
pulled the CircleCI install logs and production PackageMaze state together.

CircleCI/npm facts:

- Four install jobs completed successfully:
  - job 14 `test-node`: install step `2026-07-06T21:29:07Z` to
    `2026-07-06T21:29:57Z`, about 50s
  - job 15 `build-node`: install step `2026-07-06T21:30:16Z` to
    `2026-07-06T21:31:11Z`, about 54s
  - job 16 `test-node`: install step `2026-07-07T01:33:21Z` to
    `2026-07-07T01:34:16Z`, about 55s
  - job 17 `build-node`: install step `2026-07-07T01:34:27Z` to
    `2026-07-07T01:35:21Z`, about 54s
- Each install fetched 838 tarballs through PackageMaze, with 778 unique
  tarball URLs and 26 duplicated URL values from duplicate lockfile package
  locations.
- Job 17 timing distribution was:
  - p50 around 23.7s
  - p90 around 44.3s
  - max around 50.9s
- npm `maxsockets` defaults to 15, and the Marko repo does not override it.
  The completion curve was roughly 15-22 tarballs per second, which fits npm
  client-side queueing plus fetch time. The Prettier `50834ms` line should not
  be read as "PackageMaze spent 50s serving one request"; it likely includes
  time waiting behind npm's fetch/socket pool.

PackageMaze production evidence:

- Production D1 `usage_event_dedupe` during the rerun showed artifact download
  events completing at roughly the same 13-24 events/second as the npm logs.
- A raw R2 usage ledger sample for
  `usage_event_b46d956bb4f24d7995b11fcf94631eed` showed:
  - package `yargs-unparser@2.0.0`
  - `packageVersionId: null`
  - descriptor `source: "cached"`
  - `servingSource: "upstream"`
- The `source: "cached"` aggregate dimension is therefore misleading for this
  investigation. It means the intended Package Version source, not that the
  request was served from a retained/cache hit. The usage ledger's
  `servingSource` is the reliable field for cache-hit vs upstream-hit.
- Production retained artifacts for Feed
  `feed_1a344ea65c0a4b868f7aab047da4bae9` (`packagemaze/dogfood-npm`) were
  created slowly after the rerun:
  - at `2026-07-07T02:19:19Z`, only 491 retained artifacts existed
  - first retained artifact: `2026-07-07T01:33:29Z`
  - latest retained artifact at that point: `2026-07-07T02:18:57Z`
  - span: about 45 minutes
  - typical retention rate: 5-17 artifacts per minute
- `package_usage_activity_rollups` still had no rows for the Feed, because cold
  upstream metering does not have retained Package Version ids. This confirms
  the Sessions visibility issue separately from the throughput problem.

Code/config bottleneck:

- `runtime-worker/src/npm/package-routes.ts` streams a full cold npm tarball
  from upstream to npm and schedules artifact-fill work after the response.
  This is the intended hot-path shape.
- `runtime-worker/src/external-feeds/throttle.ts` sets
  `EXTERNAL_FEED_THROTTLE_MAX_CONCURRENT = 1`.
- The throttle key normalizes npmjs to one upstream-source key, so unrelated
  package fills for npmjs contend on the same Durable Object throttle.
- `infra/cloudflare/runtime/wrangler.jsonc` configures
  `packagemaze-artifact-fill` with `max_batch_size: 1` and
  `max_concurrency: 2`.
- Together, the queue and upstream throttle make the cache warmer far slower
  than npm's install demand.

Deployment note:

- The earlier `2026-07-06T21:29Z` original workflow ran before later production
  Runtime deployments at `2026-07-06T22:49Z` and `2026-07-07T01:25Z`.
- Current observed production release was `8808568b9`.
- Later PR `#1478` (`a87c2a874`) adds observed CI activity and artifact-fill
  job visibility, but it is not a throughput fix. It also awaits a D1
  fill-job upsert before returning the cold tarball response, which needs care
  because this is the same hot path under load.

Conclusion:

- CircleCI setup was not the bottleneck.
- npm's 50s tail is partly expected from npm's own fetch/socket pool on a
  large cold install.
- PackageMaze's bigger bug is that it accepts hundreds of cold upstream
  downloads but warms retained artifacts at only a few to a dozen per minute,
  so a second job/rerun remains cold.
- I filed PackageMaze issue
  `https://github.com/packagemaze/packagemaze/issues/1481` for this throughput
  and cache-warming bug.
