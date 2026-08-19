# Release pipeline and rollback

GitHub Actions, Docker Compose v2, Next.js 16.3, Node 20.9 or later, against a
Django and DRF backend. This file owns the path from a merged commit to a
serving process, and the path back. The subjects are the pipeline that produces
the artifact, the values that differ between two environments, and the
replacement of a running release. They also include the rollback, and the order
of a deploy against a backend change.

The build output, the container image, and the account inside it are
`references/build-output-and-container-image.md`. The supervisor, the reverse
proxy, and the environment file are
`references/runtime-process-and-reverse-proxy.md`. The order of the gates that
run before a release is `references/merge-gates-and-quality-signals.md`.

## Principle

A deploy that nobody can reverse is a decision that nobody can correct. Build
the way back before you take the way forward, and run it once against the real
system.

A deploy is finished when a request proves it. A command that returns proves
that the command returned.

One artifact reaches every environment. Where each environment builds its own,
the build that passed the tests is not the build that serves the users.

Two systems that release on their own schedules must change in an order where
each pair of versions works. A change that needs both sides at once is a change
that needs an outage.

A pipeline holds the credentials that reach production, and it executes code on
every push. Treat it as a production system, and not as a script.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The pipeline is a production system

```yaml
# Wrong: a moving tag on a third-party action, and no permission block.
# Failure: the action owner moves the tag to a new commit, and the next run
# executes code that nobody reviewed. The default token carries write access to
# the repository, so that code pushes a commit and reads every secret.
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: some-org/ship-it@v3
```

```yaml
# Correct: each action is pinned to a commit, and the job states its scope.
permissions:
  contents: read
  id-token: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # Read each identifier from the release that you audited. The two below
      # are placeholders, and a run against them fails.
      - uses: actions/checkout@0000000000000000000000000000000000000000   # v4.2.2
      - uses: some-org/ship-it@1111111111111111111111111111111111111111   # v3.2.1
```

A tag is a pointer, and the owner of the repository moves it. A commit
identifier is the content. Pin every third-party action to a full commit
identifier, and write the version in a comment beside it.

State a `permissions` block. The default token of a workflow carries more access
than a build needs, and a job that only reads the source needs
`contents: read`.

NEVER run the code of a fork under `pull_request_target`. That trigger gives the
job the secrets of the base repository, and the checked-out code is the code of
the person who opened the request.

Take a short-lived credential over a stored one. `id-token: write` lets the job
exchange the identity token of the run for access at the registry. No long-lived
key then sits in the settings of the repository.

`references/secret-boundary-and-supply-chain.md` owns the judgment of a package
and of an advisory, and it holds a veto.
`references/dependencies-and-git-workflow.md` owns the install policy that the
pipeline runs. `references/merge-gates-and-quality-signals.md` owns which stages
run, in which order, and which of them a branch requires.

### One artifact reaches every environment

```tsx
// Wrong: a browser value that must differ between two environments.
// Failure: `next build` inlines the staging address into the client bundle. The
// image that passed the staging tests then calls staging from production, and
// no environment change repairs it. Only a new build does.
"use client"; // reason: the widget runs in the browser
import { env } from "@/env";

export function Uploader() {
  return <UploadWidget endpoint={env.NEXT_PUBLIC_UPLOAD_ORIGIN} />;
}
```

```tsx
// Correct: a Server Component reads the value when the request runs.
import { connection } from "next/server";
import { env } from "@/env";

export default async function UploaderPanel() {
  await connection();
  return <Uploader endpoint={env.UPLOAD_ORIGIN} />;
}
```

`next build` replaces every `NEXT_PUBLIC_` read with its value. The value is
then a literal inside a chunk, and the environment of the host never reaches it.

`await connection()` marks the render as a request, so the read happens at
request time. The same image then serves staging and production, and each one
supplies its own value.

A value that the browser needs has two correct answers. A Server Component reads
it and passes the field to the client leaf, as the sample above does. Where the
browser must hold the value itself, accept one build for each environment, and
state that decision.

`references/app-router-structure.md` owns the `NEXT_PUBLIC_` prefix and the rule
that a variable without it stays on the server.
`references/secret-boundary-and-supply-chain.md` owns the consequence of a
credential in such a variable, and it holds a veto.
`references/boundary-validation-and-api-types.md` owns the parse that produces
the `env` object above.

### The replacement is health-checked before it takes the traffic

```bash
# Wrong: the deploy stops the running release before the new one answers.
# Failure: every request between the stop and the first healthy response ends
# with 502. A deploy in a busy hour is an outage, and the deploy log records it
# as a success.
docker compose up -d --force-recreate frontend
```

```bash
# Correct: start first, probe, smoke run, switch, then stop the previous one.
docker compose up -d --no-deps frontend-next
until curl -sf "$NEXT_ORIGIN/api/health"; do sleep 2; done
curl -sf "$NEXT_ORIGIN/" > /dev/null
ln -sfn /etc/nginx/upstreams/next.conf /etc/nginx/upstreams/active.conf
nginx -s reload
docker compose stop frontend-previous
```

The probe answers for the chain, and not for the process.
`references/degradation-and-health-checks.md` owns that route, its timeout, and
the rule that it calls Django.

The probe is not the whole proof. Add a smoke run against a real route, and
assert on the body rather than on the status. A process that starts and renders
an error page passes a status check.

The switch is one reload of the proxy, which loses no connection in flight.
`references/runtime-process-and-reverse-proxy.md` owns the upstream block that
the switch selects.

### The rollback is one command, and somebody has run it

Keep the previous image, and keep the upstream file that names it. A rollback
that needs a build is a rollback that arrives after the incident.

```bash
# Correct: the rollback is the same switch, in the other direction.
ln -sfn /etc/nginx/upstreams/previous.conf /etc/nginx/upstreams/active.conf
nginx -s reload
curl -sf "$APP_ORIGIN/api/health"
```

Run that command against production once, at a quiet hour, before you depend on
it. A path that nobody has executed is a plan, and not a rollback.

CAUTION: a rollback of the frontend alone does not undo a migration. Where the
release changed the schema, the backend owns the reverse, and the order below is
what keeps the two independent.

### More than one instance changes three answers

```ts
// next.config.ts
// Correct: every instance reads one cache, and none of them keeps its own.
import path from "node:path";
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandler: path.resolve("./cache-handler.js"),
  cacheMaxMemorySize: 0,
};

export default nextConfig;
```

```ts
// Wrong: the proxy carries two upstreams, and the configuration states neither
// key.
// Failure: `revalidateTag()` runs on the instance that served the write. The
// second instance still holds the previous answer, so one reader sees the new
// value and the next reader sees the old one.
const nextConfig: NextConfig = {
  // the default cache is per-instance, in memory and on the local disk
};
```

The default revalidation cache lives in the memory and on the disk of one
instance. A shared handler moves it to a store that every instance reads. Set
`cacheMaxMemorySize` to 0, so no instance keeps a copy in front of that store.

A handler must implement `refreshTags()`, so a tag that one instance revalidates
reaches the others. Next 16 also exposes a `cacheHandlers` key for the
`"use cache"` directive, and a multi-instance deployment sets it for the same
reason.

Two more values must be equal across the instances of one release.
`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` decrypts a Server Action payload, and
`deploymentId` tells a client which build it holds. The pipeline supplies both,
and it supplies one value to every instance and to every deploy of one release
line. `references/degradation-and-health-checks.md` owns what each key repairs
for an open tab.

A single instance with a persistent local disk needs none of this. Reach for the
handler at the second instance, and not before.

### The contract changes in two steps

A frontend release and a backend release do not land at the same instant. Order
them so that each pair of versions works.

1. Deploy the backend change that adds. A new field, a new endpoint, and a
   widened type each leave the current frontend working.
2. Regenerate the client types, and deploy the frontend that reads the new
   shape.
3. Deploy the backend change that removes. The old field leaves only after no
   client reads it.

A release that reverses this order breaks the client for the period between the
two deploys. A release that performs both steps at once has no period in which
a rollback of one side is safe.

`references/openapi-schema-and-codegen.md` owns the schema, the generator, and
the drift gate that fails a build on a renamed field. The backend owns the
classification of a change as breaking or additive. This file owns the order in
which the two deploys run.

### A non-production deployment differs in its values, and never in its code

The pipeline supplies a different environment value to staging, and the
application reads that value. It never reads the host of the request, and it
never carries a branch that only staging takes.

`references/crawl-and-index-control.md` owns the robots rules and the
`X-Robots-Tag` header that such a deployment sends, and the environment gate
that selects them. The smoke run after a non-production deploy requests
`/robots.txt` and reads that header.

### The libraries

The table gives each tool its rule and its maintenance status. Read the
installed version before you write a workflow file.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | GitHub Actions | The reference pipeline for this stack. Pin every third-party action to a commit, and state a `permissions` block. | Hosted | — | GitHub, active | — |
| Recommend | An OIDC exchange for a registry credential | Every pipeline that pushes an image. It replaces a long-lived key in the settings of the repository. | Hosted | — | GitHub, active | — |
| Recommend | One build identifier, supplied by the pipeline | Every deployment. The pipeline is the only place that gives every instance and every deploy of a release line the same value. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Conditional | A shared `cacheHandler`, over Redis | Only at two or more instances, or on a disk that does not survive a restart. It adds a dependency and a `refreshTags()` implementation. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Conditional | `cacheHandlers` | Only where the application uses `"use cache"` and runs more than one instance. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Conditional | A managed platform, over a host of your own | A middle path. It removes the proxy and the supervisor from your work, and it removes the control with them. | — | — | — | — |
| Reject | `docker compose restart`, or a forced recreation, as the deploy | It stops the running release before the new one answers, so every request in that period fails. | — | — | — | — |
| Reject | A moving tag on a third-party action | The owner moves the tag, and the next run executes code that nobody reviewed. | — | — | — | — |
| Reject | `pull_request_target` that checks out the code of a fork | The job then runs untrusted code with the secrets of the base repository. | — | — | — | — |
| Reject | A rollback that needs a build | It arrives long after the incident that it repairs. | — | — | — | — |
| Reject | A `NEXT_PUBLIC_` value that a promotion is expected to change | The build inlined it, and only a new build changes it. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| Every deploy drops requests | The deploy recreates the container in place | Send traffic through a deploy and count the 502 responses | Start the new release first, probe it, then switch |
| A deploy reports success and the site is broken | No smoke run followed the start | Read the deploy script for a request after the start | Probe the health route, then make a smoke run against one real route |
| The production build calls the staging backend | A `NEXT_PUBLIC_` value was baked at build time | Compare the value in the chunk against the environment of the host | Read the value in a Server Component, or build for each environment |
| A rollback takes an hour | No previous image is retained | Read which tags the registry and the host still hold | Keep the previous tag, and switch the upstream |
| "Failed to find Server Action" on some requests | The action encryption key differs between instances | The error appears only behind two or more upstreams | Supply one key to every instance of the release line |
| One instance serves new content and another serves old | Each instance keeps its own revalidation cache | Revalidate a tag, then request the route from each instance | Share a `cacheHandler`, and set `cacheMaxMemorySize` to 0 |
| A client breaks in the window between two deploys | The backend removed a field before the frontend stopped reading it | Compare the two release times against the schema change | Add, deploy, migrate the client, then remove |
| A run executes code that nobody reviewed | A third-party action moved its tag | Read the workflow for a tag where a commit belongs | Pin every action to a commit identifier |
| A secret leaves through a pull request | `pull_request_target` checked out the code of a fork | Read the trigger and the checkout reference | Use `pull_request`, and never check out fork code with secrets |
| The staging origin appears in the search results | The pipeline supplied the production environment value | Request the robots file of the staging origin | Supply the staging value, and read the header in the smoke run |

### Version discipline

Read the installed versions before you write a workflow file.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 makes Turbopack the default bundler | A pipeline step that passes `--turbopack` to the build | Remove the flag from the workflow |
| Next 16 exposes `cacheHandlers` beside `cacheHandler` | A multi-instance deployment that uses `"use cache"` and states only the older key | Set both keys, and implement `refreshTags()` |
| The July 2026 security release set the floor at 16.2.11 and 15.5.21 | A pipeline that ships a version under the floor of its line | Raise the version, and fail the pipeline under it |
| Next 16.2 made the Adapter API stable | A hand-written deployment step for a platform that publishes an adapter | Set `adapterPath`, or `NEXT_ADAPTER_PATH` |
| Next 16.3 opts Instant Navigations in through `cacheComponents` and `partialPrefetching` | A release note that claims the behavior with neither flag set | Read `next.config.ts` before you claim the behavior |

## Verification

```bash
# 1. Read the installed version before you write a workflow file.
rg -n '"next"' package.json

# 2. Find a third-party action that a tag names. Read every hit.
rg -n 'uses:' .github/workflows/

# 3. Find a workflow with no permission block. This prints nothing.
rg --files-without-match 'permissions:' .github/workflows/

# 4. Find a workflow that runs fork code with secrets. This prints nothing.
rg -n 'pull_request_target' .github/workflows/

# 5. Confirm the deploy strategy. Read the hit.
rg -n 'docker compose|nginx -s reload|health' deploy.sh

# 6. Find a deploy that recreates in place. This prints nothing.
rg -n 'force-recreate|compose restart' deploy.sh

# 7. Confirm that the deploy probes before it switches.
rg -n -B2 -A2 'api/health' deploy.sh

# 8. Confirm that a rollback path exists, and read it.
rg -n -A6 'rollback' deploy.sh

# 9. Run the rollback against production once, then confirm the answer.
./deploy.sh rollback && curl -sf "$APP_ORIGIN/api/health"

# 10. Confirm that the pipeline supplies one identifier to every instance.
rg -n 'GIT_SHA|NEXT_SERVER_ACTIONS_ENCRYPTION_KEY' .github/workflows/ deploy.sh

# 11. At two or more instances, confirm the shared cache keys.
rg -n 'cacheHandler|cacheHandlers|cacheMaxMemorySize' next.config.ts

# 12. Revalidate one tag, then read the route from each instance. Expect one
#     answer from both.
curl -s "$INSTANCE_A/products/1" | rg 'data-revision'
curl -s "$INSTANCE_B/products/1" | rg 'data-revision'

# 13. Find a browser value that a promotion is expected to change. Read each
#     hit against the environment of the host.
rg -n 'NEXT_PUBLIC_' .env.production .env.staging

# 14. In the smoke run of a non-production deploy, confirm the crawl refusal.
curl -s "$STAGING_ORIGIN/robots.txt" | rg 'Disallow: /'
```

## Review checklist

- [ ] Does the pipeline build the artifact once, and does every environment
      receive that artifact?
- [ ] Is every third-party action pinned to a commit identifier, with the
      version in a comment?
- [ ] Does every workflow state a `permissions` block?
- [ ] Does no workflow run the code of a fork under `pull_request_target`?
- [ ] Does the pipeline exchange a short-lived credential rather than hold a
      long-lived key?
- [ ] Does the new release start and answer before the previous one stops?
- [ ] Does the deploy probe the health route, and then make a smoke run against
      one real route?
- [ ] Does the rollback take one command, and has somebody run it against
      production?
- [ ] Does the host still hold the previous image and its upstream file?
- [ ] Does every value that must differ between environments reach the process
      at run time, rather than through a `NEXT_PUBLIC_` read?
- [ ] Does each environment supply its own value to one image?
- [ ] At two or more instances, do `cacheHandler` and `cacheMaxMemorySize`
      appear, and does the handler implement `refreshTags()`?
- [ ] Do every instance and every deploy of one release line share the action
      encryption key and the build identifier that the pipeline supplies?
- [ ] Does a contract change deploy as add, then client, then remove?
- [ ] Does the client regenerate its types in the same release that reads the
      new shape?
- [ ] Does a non-production deploy differ from production in its values alone?

## Handoffs

- The build output, the container image, the base image, and the memory that a
  build needs → `references/build-output-and-container-image.md`.
- The supervisor, the restart policy, the memory limit, the environment file,
  the reverse proxy, and the upstream that a deploy switches →
  `references/runtime-process-and-reverse-proxy.md`.
- Which stages run, in which order, and which of them a branch requires →
  `references/merge-gates-and-quality-signals.md`.
- The lockfile, the install policy, and the git hooks →
  `references/dependencies-and-git-workflow.md`.
- The judgment of a package, of an advisory, and of the security floor of the
  framework → `references/secret-boundary-and-supply-chain.md`. That domain
  holds a veto.
- The `NEXT_PUBLIC_` prefix, and `next.config.ts` as a file →
  `references/app-router-structure.md`.
- The parse over the environment, and the schema that produces the `env` object
  → `references/boundary-validation-and-api-types.md`.
- The schema, the generator, and the drift gate that fails a build on a renamed
  field → `references/openapi-schema-and-codegen.md`.
- The health route, its timeout, the open tab after a deploy, and the two skew
  keys → `references/degradation-and-health-checks.md`.
- The robots rules of a non-production deployment, and the environment gate that
  selects them → `references/crawl-and-index-control.md`.
- The report that a released build raises, and the alert over it →
  `references/error-capture-and-reporting.md`.
- The classification of a contract change as breaking or additive → the
  backend. The reverse of a schema change → the backend. The Django process, its
  go-live gate, and its own deploy → the backend. This file owns the order of
  the two deploys, and the Next.js side of each one.
