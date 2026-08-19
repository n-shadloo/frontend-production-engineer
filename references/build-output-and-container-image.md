# Build output and container image

Next.js 16.3, Node 20.9 or later, pnpm 10.x, Docker with BuildKit. This file
owns the artifact that a release ships. The subjects are the build output mode,
the memory that a build needs, and the machine that runs it. They also include
the container image, the account inside it, and the signal that stops the
process.

The supervisor, the reverse proxy, and the environment file are
`references/runtime-process-and-reverse-proxy.md`. The pipeline that runs the
build, and the deploy that replaces a release, are
`references/release-pipeline-and-rollback.md`. `next.config.ts` as a file, and
the Next 16 defaults inside it, are `references/app-router-structure.md`.

## Principle

A build is a function of the source and of the toolchain. Run it on a machine
that controls both, and ship the result. A build on the target host makes the
artifact a function of that host.

An artifact that nobody can reproduce is an artifact that nobody can audit. One
commit must produce one output, on every machine that builds it.

An image carries what the process needs at run time, and nothing more. Every
other file costs storage, download time, and attack surface.

A process inside a container is one escape away from the account that started
it. Give that account no privilege.

A stop is not a kill. A process that accepted a request must answer it before
it exits.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The output mode decides every step after it

| The application | The output | What the mode costs |
| --- | --- | --- |
| It renders on the server, it revalidates, or it exports a Server Action or a Route Handler | `output: 'standalone'`, and `node server.js` | The trace holds the modules that the server needs, and nothing else. You copy `public` and `.next/static` beside the traced output yourself. |
| It needs no server at all | `output: 'export'`, to a static host | The mode disables server rendering, revalidation, Route Handlers, Server Actions, `proxy.ts`, `cookies()`, `headers()`, `draftMode()`, and the built-in image optimizer. One route that needs a server ends the mode. |
| It targets a platform that publishes an adapter | `adapterPath`, or `NEXT_ADAPTER_PATH` | The platform owns the shape of the output. A host behind your own reverse proxy needs no adapter. |

The default build ships the whole dependency tree. `output: 'standalone'` traces
the modules that the server loads, and it writes a `server.js` entry point
beside them. The traced folder is a fraction of the size of `node_modules`.

The trace is not the whole artifact. `public` and `.next/static` stay where the
build wrote them, and the runner copies both. This is the most common failure of
a first standalone image.

```dockerfile
# Wrong: the runner copies the traced output alone.
# Failure: every page renders with no stylesheet and no client chunk, and every
# file under public answers 404. The container starts, and the health check
# passes, so the deploy reports success.
COPY --from=builder /app/.next/standalone ./
CMD ["node", "server.js"]
```

```dockerfile
# Correct: three copies, and the static folder keeps its path.
COPY --from=builder --chown=node:node /app/public ./public
COPY --from=builder --chown=node:node /app/.next/standalone ./
COPY --from=builder --chown=node:node /app/.next/static ./.next/static
CMD ["node", "server.js"]
```

`server.js` reads `PORT` and `HOSTNAME` from the environment. Set both. The
standalone server does not bind by default to the address that a container
needs.

### The build runs on the pipeline machine

```bash
# Wrong: the deploy script compiles on the target host.
# Failure: the build needs more memory than a 1 GB host holds, and the kernel
# sends SIGKILL. The compile also runs against whatever toolchain that host
# carries, so the artifact differs from the one that the tests passed against.
ssh deploy@app-host 'cd /opt/app && git pull && pnpm build && systemctl restart nextjs'
```

```bash
# Correct: the pipeline builds the image, and the host receives it.
docker build --tag "$REGISTRY/app:$(git rev-parse --short HEAD)" .
docker push "$REGISTRY/app:$(git rev-parse --short HEAD)"
```

V8 gives the old generation about 4 GB on a 64-bit machine. A build that passes
that number on a small host reaches the out-of-memory killer of the kernel. The
log then reports `JavaScript heap out of memory`, or a bare SIGKILL.

CAUTION: a build on the target host competes for memory with every other process
on it. Where such a build is unavoidable, set
`NODE_OPTIONS=--max-old-space-size` to about 75 percent of the memory of the
host. Treat that setting as a repair, and move the build to the pipeline.

`references/release-pipeline-and-rollback.md` owns the workflow file that runs
the build. `references/merge-gates-and-quality-signals.md` owns the order of the
gates that run before it.

### The image is multi-stage, and the runner is the last stage

```dockerfile
# Correct: three stages, and the runner holds no toolchain.
ARG NODE_VERSION=24.13.0-slim
FROM node:${NODE_VERSION} AS dependencies
WORKDIR /app
RUN corepack enable pnpm
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

FROM node:${NODE_VERSION} AS builder
WORKDIR /app
RUN corepack enable pnpm
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production
RUN pnpm build

FROM node:${NODE_VERSION} AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME=0.0.0.0
RUN mkdir .next && chown node:node .next
COPY --from=builder --chown=node:node /app/public ./public
COPY --from=builder --chown=node:node /app/.next/standalone ./
COPY --from=builder --chown=node:node /app/.next/static ./.next/static
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

```dockerfile
# Wrong: one stage, the whole dependency tree, and the root account.
# Failure: the image passes 1 GB, and it carries the compiler toolchain and
# every development dependency as attack surface. The process runs as root, so
# one escape owns the container.
FROM node:24
WORKDIR /app
COPY . .
RUN npm install && npm run build
CMD ["npm", "start"]
```

The `node` image carries a `node` account, so the runner needs no account of its
own. `USER node` must follow the copies, because a copy runs as the current
account of the stage.

The runner creates `.next` and gives it to the `node` account before the copies.
The framework writes its cache under that folder at run time, and a folder that
root owns refuses the write.

The install stage takes a BuildKit cache mount, so a rebuild with an unchanged
lockfile reuses the store. `pnpm install --frozen-lockfile` fails where the
lockfile and `package.json` disagree, which is the rule that
`references/dependencies-and-git-workflow.md` states for every install.

Pin the base image to an exact patch version, and keep that version at or above
the Node floor of the project. `references/dependencies-and-git-workflow.md`
owns the agreement between `packageManager`, `engines.node`, and `.nvmrc`, and
the base image is the fourth place that states a Node version.

### The base image is glibc, unless size decides otherwise

The slim image carries glibc and about 226 MB. The Alpine image carries musl and
about 110 MB. The official container example ships the slim image, and it
documents the switch.

Take slim by default. A native module publishes a prebuilt binary for glibc more
often than for musl, and `sharp` is the module that this stack depends on.

Take Alpine only where the image size is a stated constraint, and only after a
test proves every native module inside the built image.

### The image optimizer needs `sharp` inside the image

The self-hosting guide states that the image optimizer needs no configuration
on a host of your own. Reports against standalone builds and monorepo builds
describe a missing `sharp` module at run time, so the trace does not always
carry it.

Add `sharp` to `dependencies`, and request one optimised image from the built
container before the release. A missing module answers with a 500, or it serves
the source file with no optimisation.

`references/image-and-video-delivery.md` owns the props of `<Image>`, the
`remotePatterns` list, and the decision to hand the optimizer to a content
network instead.

### The build context carries the source alone

```text
# Correct: .dockerignore, with the entries this stack needs.
node_modules
.next
.git
Dockerfile
.dockerignore
.env*.local
npm-debug.log
```

A project with no `.dockerignore` sends the whole working folder to the build.
The local `node_modules` then overwrites the installed one, and the local
`.next` enters the image. `.env.local` carries every development secret into a
layer that anybody who pulls the image reads.

`references/secret-boundary-and-supply-chain.md` owns the rule that a secret
never reaches an artifact that others read. This file owns the file that keeps
it out of the build context.

### The process receives the signal

`node server.js` handles `SIGTERM` and `SIGINT`. It stops accepting connections,
it finishes the requests that it holds, and it runs the `after()` callbacks that
those requests registered. Allow 10 to 30 seconds for that period.

```dockerfile
# Wrong: the shell form of CMD.
# Failure: /bin/sh runs as PID 1 and it forwards no signal. A stop waits out the
# whole grace period, then sends SIGKILL, and every request in flight ends with
# no response.
CMD npm start
```

```dockerfile
# Correct: the exec form makes the server PID 1, which receives the signal.
CMD ["node", "server.js"]
```

Add `dumb-init` or `tini` only where a wrapper takes PID 1 and the signal stops
there. The exec form needs neither.

`references/runtime-process-and-reverse-proxy.md` owns the grace period that the
supervisor grants, and the restart policy around it.

### The libraries

The table gives each package its rule and its maintenance status. Read the
installed version from `package.json` before you write code.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `output: 'standalone'` | Every deployment that serves a dynamic route from your own host. It ships with the framework. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Recommend | The `node` slim base image | The default base for this stack. It carries glibc, which every native module in it supports. | Node 24.13.0-slim | Current | Node.js, active | None |
| Recommend | `sharp` | Every self-hosted image optimizer. Add it to `dependencies`, and prove it inside the built image. | 0.34 or later | Current | Lovell Fuller, active | None |
| Conditional | `output: 'export'` | Only where no route needs a server. One `cookies()` call ends the mode. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Conditional | The `node` Alpine base image | Only where the image size is a stated constraint, and only after a test of every native module. | Node 24.11.1-alpine | Current | Node.js, active | None |
| Conditional | `dumb-init` or `tini` | Only where a wrapper holds PID 1 and stops the signal. The exec form of `CMD` needs neither. | Current | Current | Active | None |
| Conditional | `adapterPath` and `NEXT_ADAPTER_PATH` | Only where the target platform publishes an adapter. A host behind your own proxy needs none. | Next.js 16.3 | 2026-08-03 | Vercel, active | None on 16.3 |
| Audit-only | `next start` against a standalone build | `node server.js` is the entry point that the trace produces. The other command loads more than the server needs. | — | — | — | — |
| Reject | `next build` on the production host | The build competes for memory with the running processes, and the artifact becomes a function of that host. | — | — | — | — |
| Reject | `next dev` as a production process | It compiles for each request, it holds far more memory, and it exposes development behavior. | — | — | — | — |
| Reject | A single-stage image | It ships the compiler toolchain and every development dependency to production. | — | — | — | — |
| Reject | A container with no `USER` line | The process runs as root, which turns one escape into full control of the container. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| Every page renders with no stylesheet | The runner copied the trace and not `.next/static` | Request one chunk under `_next/static` and read the status | Copy `public` and `.next/static` beside the traced output |
| The image passes 1 GB | One stage, and the whole dependency tree | `docker images` reports the size, and the Dockerfile holds no `AS builder` | Multi-stage build, over `output: 'standalone'` |
| `id -u` reports 0 inside the container | No `USER` line, or one that names root | `docker run --rm <image> id -u` | Add `USER node` after the copies |
| The container starts and cannot write its cache | `.next` belongs to root, and the process is the `node` account | Read the start-up log for a permission error | Create `.next` and chown it before the copies |
| `/_next/image` answers 500 | The trace carries no `sharp` | Request one optimised image from the built container | Add `sharp` to `dependencies` |
| A native module fails only in the image | The Alpine base carries musl, and the module ships a glibc binary | Compare the base image against the local machine | Take the slim base image |
| The build is killed with no error message | The heap or the host memory ran out | The log reports `JavaScript heap out of memory`, or the kernel log reports the killer | Build on the pipeline machine |
| The deploy drops every request in flight | The shell form of `CMD` holds PID 1 | A stop takes the whole grace period | Take the exec form of `CMD` |
| A secret is inside an image layer | No `.dockerignore`, so `.env.local` entered the context | Read the layer contents of the built image | Add `.dockerignore`, then rotate the secret |
| The image behaves differently from the tested build | The local `node_modules` entered the context | Read `.dockerignore` for the entry | Ignore `node_modules` and `.next` |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 makes Turbopack the default bundler | A build script that passes `--turbopack` | Remove the flag, and pass `--webpack` only where a webpack key remains |
| Next 16 raised the Node floor to 20.9 | A base image tag under `node:20.9`, or one that disagrees with `.nvmrc` | Pin the base image at or above the floor, and match the three other places that state it |
| Next 16.2 made the Adapter API stable | A hand-written deployment shim for a platform that publishes an adapter | Set `adapterPath`, or `NEXT_ADAPTER_PATH` |
| The official container example moved to the Node slim base | A Dockerfile that names an Alpine base with no stated reason | Take the slim base, unless a test proves every native module on musl |
| Next 16.3 lowered the memory that a long development session holds | A build host sized against the Next 15 figures | Measure again before you raise a limit |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"sharp"|"packageManager"' package.json

# 2. Confirm the output mode. Read the hit.
rg -n "output:\s*['\"](standalone|export)['\"]" next.config.ts

# 3. Build, then confirm that the trace produced an entry point.
pnpm build && ls .next/standalone/server.js .next/static

# 4. Confirm that the runner copies all three paths. Expect three hits.
rg -n 'public|\.next/standalone|\.next/static' Dockerfile

# 5. Confirm that the image runs as a non-root account. Expect a value above 0.
docker build --tag app:check . && docker run --rm app:check id -u

# 6. Confirm the exec form of CMD. Expect the bracket form.
rg -n '^CMD' Dockerfile

# 7. Find a Dockerfile with no builder stage. This prints nothing.
rg --files-without-match 'AS builder' Dockerfile

# 8. Confirm that the build context excludes the four paths.
rg -n 'node_modules|\.next|\.git|\.env' .dockerignore

# 9. Find a build that runs on the target host. This prints nothing.
rg -n 'next build|pnpm build|npm run build' deploy*.sh Makefile

# 10. Read the size of the built image. Compare it against the previous tag.
docker images app --format '{{.Tag}} {{.Size}}'

# 11. Start the image, then request one optimised image. Expect 200.
docker run --rm -d -p 3000:3000 --name check app:check
curl -s -o /dev/null -w '%{http_code}\n' 'localhost:3000/_next/image?url=%2Flogo.png&w=256&q=75'

# 12. Stop the container, and measure the period. Expect a drain, not a kill.
time docker stop check
```

## Review checklist

- [ ] Does `next.config.ts` state an output mode, and does that mode match what
      the routes need?
- [ ] Does the runner copy `public`, the traced output, and `.next/static`?
- [ ] Does the Dockerfile hold a separate dependency stage and builder stage?
- [ ] Does the runner stage hold no compiler toolchain and no development
      dependency?
- [ ] Does a `USER` line follow the copies, and does it name a non-root account?
- [ ] Does the runner create `.next` and give it to that account?
- [ ] Does the base image name an exact patch version at or above the Node floor
      of the project?
- [ ] Does the install run with `--frozen-lockfile`?
- [ ] Does `sharp` sit in `dependencies`, and does one request prove the
      optimizer inside the built image?
- [ ] Does `.dockerignore` exclude `node_modules`, `.next`, `.git`, and every
      local environment file?
- [ ] Does `CMD` take the exec form, so the server holds PID 1?
- [ ] Does the build run on the pipeline machine rather than on the target host?
- [ ] Does the project state a memory setting only where a build on the host is
      unavoidable?
- [ ] Does the image carry `PORT` and `HOSTNAME`?

## Handoffs

- The supervisor, the restart policy, the memory limit, the environment file,
  and the reverse proxy in front of the process →
  `references/runtime-process-and-reverse-proxy.md`.
- The workflow file that builds the image, the deploy that replaces a release,
  and the rollback → `references/release-pipeline-and-rollback.md`.
- `next.config.ts` as a file, the Next 16 key changes inside it, and the
  `NEXT_PUBLIC_` prefix → `references/app-router-structure.md`.
- The lockfile, `--frozen-lockfile`, `packageManager`, `engines.node`, and
  `.nvmrc` → `references/dependencies-and-git-workflow.md`.
- The order of the gates that run before the build →
  `references/merge-gates-and-quality-signals.md`.
- The props of `<Image>`, the `remotePatterns` list, and the content network in
  front of the optimizer → `references/image-and-video-delivery.md`.
- The rule that a secret never reaches an artifact that others read, and the
  judgment of a package or an advisory →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The bytes that a route ships, and the budget over them →
  `references/performance-budgets-and-measurement.md`.
- The Gunicorn or ASGI process, its worker count, and `ALLOWED_HOSTS` → the
  backend. This file owns the Next.js artifact and the image around it.
