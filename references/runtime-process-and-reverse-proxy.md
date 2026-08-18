# Runtime process and reverse proxy

Next.js 16.3, Node 20.9 or later, Nginx, systemd, Docker Compose v2. This file
owns the process that serves the application, and the layer in front of it. The
subjects are the supervisor, the restart policy, the memory limit, and the
environment file. They also include the server block that routes to Node and to
Django, and the layer that emits a response header.

The build output, the container image, and the signal inside it are
`references/build-output-and-container-image.md`. The pipeline, the deploy, and
the rollback are `references/release-pipeline-and-rollback.md`. The value of a
security header, and the policy inside it, are
`references/security-headers-and-csp.md`.

## Principle

A process that a person starts by hand is a process that ends with the session
of that person. A supervisor owns the lifetime, and it restarts the process
after a crash.

One front door. Two layers that both emit one header emit two values, and the
reader receives one of them by chance.

Static bytes belong to the layer that is built to serve them. A request that a
proxy answers never costs the application process anything.

A host has a fixed amount of memory, and the kernel decides which process dies
when that memory runs out. Decide it first, and state the limit.

A header on an inbound request is input from outside the program. The proxy is
the place that removes the input that the application must never read.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The supervisor owns the lifetime

| The supervisor | Take it when | What it gives |
| --- | --- | --- |
| systemd | One host, and a stack that is not containerised. This is the default. | `Restart=always`, `EnvironmentFile`, `MemoryMax`, the journal, and no new dependency |
| Docker Compose v2 | The whole stack already runs in containers beside Django, Postgres, and Redis | One network, one service name for each process, and a health condition on the start order |
| PM2 | Nothing new. It is alive only in legacy code. | A second supervisor above the first one, and a dependency that the operating system already replaces |

```ini
# Correct: /etc/systemd/system/nextjs.service
[Unit]
Description=Next.js frontend
After=network-online.target

[Service]
User=nextjs
WorkingDirectory=/opt/app
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5
TimeoutStopSec=30
EnvironmentFile=/opt/app/.env.production
Environment=NODE_ENV=production PORT=3000 HOSTNAME=127.0.0.1
MemoryMax=512M
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/app/.next/cache
StandardOutput=journal
StandardError=journal
SyslogIdentifier=nextjs

[Install]
WantedBy=multi-user.target
```

```ini
# Wrong: no restart, and the secrets sit in the unit.
# Failure: one crash leaves the site down until a person notices. The unit file
# is world-readable, so every account on the host reads the service token and
# the action encryption key.
[Service]
ExecStart=/usr/bin/node server.js
Restart=no
Environment=DJANGO_SERVICE_TOKEN=8f2c1d94
Environment=NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=b7a0e3f5
```

`HOSTNAME=127.0.0.1` binds the process to the loopback interface, because Nginx
sits on the same host. A container binds to `0.0.0.0` instead, because the proxy
reaches it across the network of the container.

`TimeoutStopSec` must cover the drain of the server.
`references/build-output-and-container-image.md` states the 10 to 30 second
period that `node server.js` needs.

`MemoryMax` bounds the process. Without it the kernel chooses the process that
dies, and on a small host it often chooses Postgres.

A Compose deployment states the same three facts as `restart: unless-stopped`,
`mem_limit`, and `stop_grace_period`.

### The environment reaches the process from a file

`EnvironmentFile` names a file with mode 600, owned by the account that runs the
service. A Compose deployment names the same file under `env_file`.

NEVER commit that file. `references/dependencies-and-git-workflow.md` owns the
ignore rules that keep it out of the repository, and
`references/secret-boundary-and-supply-chain.md` owns the rule that a secret
never crosses to the browser.

Commit `.env.example` beside it, with every key and no value. A key that only
the production host holds is a key that the next deployment forgets.

Read every value once at start-up, and fail the start where one is absent. A
missing value that a route discovers is an outage that begins hours after the
deploy.

`references/boundary-validation-and-api-types.md` owns the parse over the
environment, and it names the schema that performs it.

### One server block routes to two upstreams

```nginx
# Correct: one origin, and two upstreams behind it.
map $http_upgrade $connection_upgrade { default upgrade; '' close; }

upstream nextjs { server 127.0.0.1:3000; keepalive 64; }
upstream django { server unix:/run/gunicorn/app.sock; keepalive 32; }

server {
  listen 443 ssl;
  http2 on;
  server_name app.example.com;

  ssl_protocols TLSv1.2 TLSv1.3;
  # The application emits the security header set. Add none of it here.

  # Remove the forged header of the proxy bypass before Node reads it.
  proxy_set_header x-middleware-subrequest "";

  client_max_body_size 50m;

  location /_next/static/ {
    proxy_pass http://nextjs;
    add_header Cache-Control "public, max-age=31536000, immutable";
  }

  location /api/   { proxy_pass http://django; include proxy_params; }
  location /admin/ { proxy_pass http://django; include proxy_params; }

  location /ws/ {
    proxy_pass http://nextjs;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_read_timeout 86400s;
    proxy_buffering off;
  }

  location / {
    proxy_pass http://nextjs;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;
  }
}
```

```nginx
# Wrong: the WebSocket location repeats the basic proxy block.
# Failure: the handshake never reaches 101, and the browser reports 400 or 426.
# Every live surface in the application fails at once, and the access log shows
# a plain GET where an upgrade belongs.
location /ws/ { proxy_pass http://nextjs; }
```

One origin for both applications keeps every request same-origin. The cookie
then needs no `Domain` attribute, and the browser sends no preflight. A split
across two subdomains needs a CORS configuration and a cookie domain, and
`references/cross-origin-and-bff-proxy.md` owns that decision.

`X-Forwarded-Proto` and `X-Forwarded-For` must reach Django. Django builds an
absolute URL from the first one, and its origin check on a write reads it.
`references/session-and-token-lifecycle.md` states that `Set-Cookie` must cross
this layer unchanged, and that the `__Host-` prefix survives it.

`client_max_body_size` defaults to 1 MB. Match it to the largest upload that the
application accepts. `references/file-upload-and-transport.md` owns the size at
which a direct upload replaces a request through the application.

### The static folder belongs to the proxy

`_next/static` holds files whose name carries a content hash. A new build writes
new names, so the old name never describes new bytes. Serve those files with
`public, max-age=31536000, immutable`.

Every such request that reaches `node server.js` costs the application process
time that it owes to a render. The proxy answers it for nothing.

`references/degradation-and-health-checks.md` owns the open tab that asks for a
deleted chunk after a deploy, and the two configuration keys that repair it.

CAUTION: never apply a public cache directive to a route that renders per-user
content. Next.js sends `private, no-cache, no-store, max-age=0, must-revalidate`
on a dynamic response, and an `add_header` that overrides it turns one reader's
page into every reader's page.
`references/exposed-endpoints-and-destinations.md` holds the rule, and it holds
a veto.

### The stream needs the buffer off

`proxy_buffering off` lets a streamed response leave the proxy as it arrives. A
buffered response arrives in one piece when the connection closes, so a
server-sent event feed works in development and batches in production.

The response header `X-Accel-Buffering: no` states the same instruction for one
response. `references/push-transport-and-connection.md` owns the list of what
the client depends on at this layer, and the `curl -N` command that proves it.

Partial Prerendering and a streamed Server Component depend on the same setting.
A static shell that arrives with its dynamic holes, in one piece, is a route
that lost the whole benefit of the split.

### One layer emits a response header

Choose one layer for the security header set, and record the choice. Nginx and
`next.config.ts` both emit a header when both are configured, and the response
then carries two values.

The default for this stack is the application. `proxy.ts` is the only source of
a per-request nonce, so the policy already leaves from there. A second layer
splits one decision across two files.

Take Nginx only where the application emits no header at all, and where one
value covers every response. Record that choice beside the server block.

`references/security-headers-and-csp.md` owns what each header must contain, the
nonce, and the report-only run. This file owns the rule that one layer emits it,
and the command that proves no duplicate.

`references/crawl-and-index-control.md` owns the `X-Robots-Tag` header of a
non-production deployment. Add that header at the layer that this decision
names.

### The proxy removes the bypass header

```nginx
# Correct: the header never reaches the application.
proxy_set_header x-middleware-subrequest "";
```

CVE-2025-29927 skipped `middleware.ts` completely when a request carried a
forged `x-middleware-subrequest` header. CVE-2026-64642 of July 2026 is a second
bypass of the same file. It affects an App Router build on Turbopack with one
entry in `config.i18n.locales`.

Keep the installed Next.js at 16.2.11 or later on the Active LTS line, or on the
16.3 line. `references/secret-boundary-and-supply-chain.md` owns the security
floor as a gate, and it holds a veto.

The header strip is a second layer, and never the first one. Both advisories
prove that authorization inside `proxy.ts` is not enforcement.
`references/route-protection-and-permissions.md` owns the gate in the data path
that enforces it.

### The host has a fixed amount of memory

A small host runs Node, Gunicorn, Postgres, and Redis together. State a limit
for each process, so the kernel never chooses.

The image optimizer runs `sharp` for each new transform, and each run costs
memory and processor time. On glibc set `MALLOC_ARENA_MAX=2` to hold the
fragmentation down, and cap `sharp.concurrency` where the host is small.

Three folders grow without a bound. They are the journal, the revalidation cache
under `.next/cache`, and the cache of the image optimizer. Set `SystemMaxUse`
for the journal, raise `minimumCacheTTL`, and run a periodic clean-up.

`references/image-and-video-delivery.md` owns the decision to hand the optimizer
to a content network, which removes the memory cost from the host.

### The libraries

The table gives each package its rule and its maintenance status. Read the
installed version before you write a configuration file.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | systemd | The default supervisor on one host. It ships with the operating system. | Operating system | — | Active | — |
| Recommend | Nginx | The reverse proxy in front of Node and Gunicorn. | Stable | — | Active | None |
| Recommend | certbot with Let's Encrypt | Every origin that terminates TLS itself. The renewal is a timer, and never a reminder. | Stable | — | EFF, active | None |
| Conditional | Docker Compose v2 | Only where the whole stack is already containerised. It replaces the systemd unit, and not the proxy. | v2 | Current | Docker, active | None |
| Conditional | Cloudflare in proxy mode | Only where the project needs a content network. Set the mode to Full (strict), enable the WebSockets option, and expect the 120-second read timeout. | — | — | Active | — |
| Conditional | `ngx_brotli`, or precompressed assets | Only where bandwidth is the constraint and the origin is static-heavy. | Stable | — | Active | None |
| Audit-only | PM2 | Alive in legacy installations. It adds a supervisor above the one that the host already runs. | Maintained | — | Maintained | — |
| Reject | A security header that Nginx and `next.config.ts` both emit | The response carries two values, and the browser takes one of them. | — | — | — | — |
| Reject | A public cache directive on a per-user response | The next caller receives the response of the previous one. | — | — | — | — |
| Reject | A supervisor with no memory limit on a small host | The kernel chooses which process dies, and it often chooses the database. | — | — | — | — |
| Reject | `x-middleware-subrequest` forwarded to the application | It is the header of a known bypass of `proxy.ts`. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The site stays down after one crash | The unit states `Restart=no` | Read the unit, and read the journal for the exit | Set `Restart=always` and `RestartSec` |
| The kernel kills Postgres under load | No process states a memory limit | The kernel log names the killer and the process | Set `MemoryMax` for each service |
| A WebSocket answers 400 or 426 | The location carries no upgrade headers | Read the access log for a GET where an upgrade belongs | Add the `map` block, `proxy_http_version 1.1`, and the two headers |
| A connection closes about every 60 seconds | `proxy_read_timeout` sits under the heartbeat period | Read the close code in the browser | Raise the timeout above that period |
| A stream arrives in one piece at the end | Buffering at Nginx, or at the content network | `curl -N` reports batched output | Set `proxy_buffering off`, and send `X-Accel-Buffering: no` |
| An upload answers 413 | `client_max_body_size` is under the file size | The Nginx error log reports the entity size | Raise the value to match the feature |
| The response carries two HSTS values | Both Nginx and `next.config.ts` emit it | Read the response headers of the origin | Choose one layer, and remove the other |
| One reader sees the page of another | A public cache directive on a dynamic route | Request the route with two different sessions | Remove the override, and keep the framework directive |
| Django builds an address with the wrong scheme | `X-Forwarded-Proto` never reached it | Read the redirect target that Django returns | Forward the header in the `location` block |
| A protected route answers 200 for an unknown caller | The application trusts `proxy.ts`, and the header is not stripped | Send the bypass header against a protected route | Strip the header, patch to the floor, and gate in the data path |
| The disk fills over a week | The journal and the two caches grow without a bound | `df -h`, then measure the size of `.next/cache` | Set `SystemMaxUse`, raise `minimumCacheTTL`, add a clean-up |
| The process cannot write its cache | `ProtectSystem=strict` with no writable path | Read the journal for the permission error | Add the cache folder to `ReadWritePaths` |

### Version discipline

Read the installed versions before you write a configuration file.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 serves `proxy.ts` on the Node runtime | A proxy rule that routes a request to an Edge runtime | Keep every route on the Node runtime |
| Nginx replaced the `listen ... http2` argument with the `http2` directive | `rg -n 'listen .*http2' /etc/nginx` reports a hit | Write `listen 443 ssl;` and `http2 on;` on two lines |
| The July 2026 security release fixed a second bypass of `proxy.ts` | `npx next --version` reports a version under 16.2.11 | Upgrade, and strip the header at the proxy |
| Next 16 raised the `minimumCacheTTL` default to 14400 seconds | A cache clean-up sized against the 60-second default | Read the new default before you size the disk |
| Cloudflare states a 120-second read timeout for the origin | A long request that a comment describes as a 100-second limit | Keep every request under the documented timeout |
| Next 16.3 handles more requests under one process | A host sized against a Next 15 measurement | Measure again before you add an instance |

## Verification

```bash
# 1. Read the installed version before you write a configuration file.
rg -n '"next"' package.json

# 2. Confirm the supervisor, the restart policy, and the memory limit.
systemctl cat nextjs | rg -n 'Restart=|RestartSec|TimeoutStopSec|MemoryMax|EnvironmentFile'

# 3. Confirm that the environment file is not readable by every account.
stat -c '%a %U' /opt/app/.env.production   # expect 600 and the service account

# 4. Find a secret inside the unit file. This prints nothing.
systemctl cat nextjs | rg -n 'PASSWORD|SECRET|TOKEN|_KEY='

# 5. Confirm the proxy block. Read every hit.
rg -n 'proxy_buffering|proxy_http_version|client_max_body_size|x-middleware-subrequest' /etc/nginx

# 6. Confirm that the upgrade map exists. This reports the block.
rg -n 'map \$http_upgrade' /etc/nginx

# 7. Confirm the immutable directive on a hashed asset.
curl -sI "$APP_ORIGIN/_next/static/chunks/main.js" | rg -i 'cache-control'

# 8. Confirm that no security header arrives twice. Expect one line for each.
curl -sI "$APP_ORIGIN/" | rg -i 'strict-transport-security|content-security-policy'

# 9. Confirm that an authenticated page carries no store directive.
curl -sI -H "Cookie: $SESSION_COOKIE" "$APP_ORIGIN/dashboard" | rg -i 'cache-control'

# 10. Confirm that a stream arrives in parts, and not at the close.
curl -N "$APP_ORIGIN/api/stream"

# 11. Confirm that the largest upload passes the proxy. Expect no 413.
curl -s -o /dev/null -w '%{http_code}\n' -F file=@large.bin "$APP_ORIGIN/api/upload"

# 12. Send the bypass header at a protected route. Expect a redirect or a 401.
curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'x-middleware-subrequest: proxy' "$APP_ORIGIN/dashboard"

# 13. Kill the process, then confirm that the supervisor restarts it.
sudo systemctl kill nextjs && sleep 6 && systemctl is-active nextjs

# 14. Read the memory and the disk after a week of traffic.
free -m ; df -h ; du -sh /opt/app/.next/cache
```

## Review checklist

- [ ] Does a supervisor own the process, and does it restart it after a crash?
- [ ] Does the supervisor state a stop timeout that covers the drain of the
      server?
- [ ] Does every service on the host state a memory limit?
- [ ] Does the process read its environment from a file with mode 600, owned by
      the service account?
- [ ] Is `.env.example` committed and current, with every key and no value?
- [ ] Does the start fail where a required value is absent?
- [ ] Does one server block route `/` to Node and `/api/` to Django?
- [ ] Does the proxy forward `Host`, `X-Real-IP`, `X-Forwarded-For`, and
      `X-Forwarded-Proto`?
- [ ] Does `Set-Cookie` cross the proxy unchanged?
- [ ] Does `_next/static` carry `public, max-age=31536000, immutable`?
- [ ] Does every per-user response keep the framework cache directive, with no
      public override at any layer?
- [ ] Does a WebSocket location carry the upgrade map, `proxy_http_version 1.1`,
      and both upgrade headers?
- [ ] Does `proxy_read_timeout` sit above the heartbeat period of the client?
- [ ] Is `proxy_buffering` off wherever the application streams?
- [ ] Does `client_max_body_size` match the largest upload of the application?
- [ ] Does exactly one layer emit each security header, and does the project
      record which one?
- [ ] Does the proxy set `x-middleware-subrequest` to an empty value?
- [ ] Is the installed Next.js at 16.2.11 or later, or on the 16.3 line?
- [ ] Does TLS terminate in one place, with an automatic renewal?
- [ ] Do the journal and the two caches carry a bound?

## Handoffs

- The build output, the container image, the base image, and the drain period →
  `references/build-output-and-container-image.md`.
- The workflow file, the deploy that replaces a release, the rollback, and the
  shared cache handler → `references/release-pipeline-and-rollback.md`.
- The value of each security header, the Content Security Policy, the nonce, and
  the report-only run → `references/security-headers-and-csp.md`. That domain
  holds a veto.
- The rule that a per-user response never enters a cache at any layer, and the
  size cap on a Route Handler →
  `references/exposed-endpoints-and-destinations.md`. That domain holds a veto.
- The gate in the data path that authorizes a route, and the permitted work of
  `proxy.ts` → `references/route-protection-and-permissions.md` and
  `references/app-router-structure.md`.
- The session cookie, its prefix, and the `Set-Cookie` header that must cross
  this layer → `references/session-and-token-lifecycle.md`.
- The topology decision between one origin and two subdomains, and the CORS
  exchange behind it → `references/cross-origin-and-bff-proxy.md`.
- What the client depends on at this layer for a WebSocket and for a
  server-sent event feed → `references/push-transport-and-connection.md`.
- The upload size at which a direct upload replaces a request through the
  application → `references/file-upload-and-transport.md`.
- The `X-Robots-Tag` header of a non-production deployment, and the robots rules
  beside it → `references/crawl-and-index-control.md`.
- The open tab that asks for a deleted chunk, and the health route →
  `references/degradation-and-health-checks.md`.
- The parse over the environment, and the schema that performs it →
  `references/boundary-validation-and-api-types.md`.
- The ignore rules that keep an environment file out of the repository →
  `references/dependencies-and-git-workflow.md`.
- The security floor of the framework, and the place of a secret →
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- Gunicorn, the ASGI server, the worker count, `ALLOWED_HOSTS`, and the Django
  health endpoint → the sibling skill `django-release-readiness`. This file owns
  the Next.js process and the proxy in front of it.
