# The secret boundary and the supply chain

Next.js 16.3, React 19.2.6 or later, pnpm 10.x, `gitleaks`, TypeScript 5.9. This
file owns two questions that a review answers before a release. The first is
which values may cross into the browser. The subjects are the `NEXT_PUBLIC_`
prefix, the `server-only` guard, the serialized prop, and the payload that a
render produces. The second question is which code the browser runs that this
team did not write. The subjects are the framework security floor, the advisory
triage, and the third-party script.

The endpoint that returns a payload is
`references/exposed-endpoints-and-destinations.md`. The policy that scopes a
vendor script is `references/security-headers-and-csp.md`.

## Principle

A secret that reaches the browser once is public from that moment. Its removal
from the code changes nothing; only rotation does.

The boundary between the server and the browser is a security boundary. A build
error on that boundary is cheaper than a review that has to notice a value.

Serialization carries whatever the object holds. A component that renders three
fields still ships every field that reached it.

A dependency runs with the authority of the code that imports it. The team owns
the decision to install it, and it owns the decision to keep it.

A version is a security property. A patched framework is a control, and it is the
only control against a defect inside the framework.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but in
decline, and alive only in legacy code.

### Every `NEXT_PUBLIC_` value is public forever

Next.js inlines a `NEXT_PUBLIC_` variable into the browser bundle at build time.
The value is then a literal inside a chunk that anybody downloads.

Review every such variable the way you review a public API. Ask one question of
each: is this value safe on a page that an attacker reads at leisure? A value
that fails the question is a secret, and it belongs behind the server boundary.

`references/app-router-structure.md` states the rule and owns the prefix. This
file states the consequence and the recovery. Once the build shipped, deletion of
the line does not help. Rotate the credential, then remove the variable.

Scan the built output, and not the source alone. A secret reaches a chunk through
a barrel file or through a shared configuration module, and a grep over `src/`
misses both.

### The prop and the payload carry what you put in them

```tsx
// Wrong: the configuration object crosses the boundary whole.
// Failure: the server serializes every field of config into the payload that
// the browser receives. The widget renders the store name, and the API secret
// travels beside it in readable text.
<ClientWidget config={{ storeId: config.storeId, apiSecret: process.env.API_SECRET }} />
```

```tsx
// Correct: the module is server-only, and the prop carries one field.
import "server-only"; // the build fails if a client module imports this file

<ClientWidget config={{ storeId: config.storeId }} />;
```

Three rules follow. Import `server-only` in every module that reads a secret, so
the boundary fails at build time rather than in review;
`references/server-and-client-components.md` owns that guard. Pass fields, never
records. Read the payload of a rendered route and confirm what it holds.

A payload leak is silent. No component renders the field, no test fails, and the
value sits in the response of every reader.

### The taint APIs are a second layer

React ships `experimental_taintObjectReference` and
`experimental_taintUniqueValue`. They mark an object or a value, and React then
refuses to serialize it across the boundary.

Both are experimental, and both need `experimental.taint: true` in
`next.config.ts`. Take them as a second layer over the rules above. NEVER take a
taint call as the only protection against a leak, because the flag can be off and
the API can change.

### The framework version is the security floor

Read two versions before a release, and read them from the installed tree.

```bash
# Wrong: the review reads the range that package.json declares.
# Failure: a caret range says ^19.2.0, and the lockfile resolved 19.2.3. The
# review reports a patched application, and the installed tree carries the
# vulnerable runtime.
rg -n '"react"|"next"' package.json
```

```bash
# Correct: read the versions that the install produced.
node -p "require('next/package.json').version"
npm ls react react-dom
```

Next.js bundles its own React Server Components runtime. A React advisory
therefore reaches an application through the Next.js version, and not only
through the `react` line of `package.json`. An application that reads `react` in
`package.json` and concludes that it is safe has read the wrong number.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states. That file also records the advisory
family behind the floor, which produces a new entry often.

Next.js publishes a security release every month. Apply it within days.
`references/app-router-structure.md` records the July 2026 release, which fixed
nine defects on the 16 line.

Two reference points set the urgency. CVE-2025-29927 of March 2025 carries a
CVSS score of 9.1, and it skips the authorization file completely.
CVE-2025-55182 of December 2025 carries a CVSS score of 10.0. It is
unauthenticated remote code execution in the deserialization of a React Server
Components payload. Attackers used it in the wild within days of its
disclosure.

### The advisory triage

Run the audit in CI, and fail the job above a stated severity.

An audit reports a package, not a risk. Triage each report on two axes.

| Axis | The question | What it changes |
| --- | --- | --- |
| Severity | What does the advisory allow? | It sets the deadline |
| Reachability | Does this application call the affected code path? | It sets the priority, never the deadline of a critical defect |

A high-severity advisory in a package that only the test run imports is still
work, and it is not an incident. A moderate advisory in a package that renders
user content is the opposite. Record the judgment beside the exception, so the
next reader does not repeat the analysis.

NEVER dismiss an audit report with no record. OWASP Top 10:2025 adds software
supply chain failures as its own category, and an ignored report is the plain
form of that failure.

`references/dependencies-and-git-workflow.md` owns the install policy, which is
`minimumReleaseAge`, the allowlist of packages that may run a lifecycle script,
and the update bot. This file owns the judgment of whether a package is
dangerous.

### A third-party script is code that somebody else deploys

| Condition | Action | The cost |
| --- | --- | --- |
| The script can be served from the origin of the application | Self-host it | The team owns the update of the file |
| The script must load from a vendor origin, and the vendor publishes a fixed file | Add an `integrity` attribute and a `crossorigin` attribute | The attribute must change when the vendor publishes a new file |
| The vendor rewrites the file, so no hash is stable | Scope it in the policy, and give it the smallest surface | A vendor change reaches every reader with no review |

```tsx
// Correct: the vendor file is pinned by hash, and the browser refuses a change.
import Script from "next/script";

<Script
  src={widgetSrc}
  integrity="sha384-THE_HASH_THAT_THE_VENDOR_PUBLISHES"
  crossOrigin="anonymous"
  strategy="lazyOnload"
/>;
```

A tag manager is a channel through which somebody adds code after the review.
Treat it as such, and hold the record of who may publish through it.

`references/client-bundle-and-third-party-scripts.md` owns the strategy, the
measured cost, and the named owner of each script.
`references/consent-gate-and-cookie-inventory.md` owns the consent that must
arrive before a tag renders. That file states that the gate covers the container,
and never what somebody publishes into it. This file owns the integrity of the file
and the decision to self-host.

### The secret that reaches a commit

Run a secret scanner in CI, and in the pre-commit hook.
`references/dependencies-and-git-workflow.md` owns the hook and the command,
which is `gitleaks git`. This file states the response.

A secret that reached a commit is compromised, whatever the history now shows.
Rotate it first. Rewrite the history second, and only where the repository allows
it. A commit that a fork or a clone already carries is beyond recall.

Scan two things that a source scan misses. Scan the built bundle, because a
secret can reach a chunk through a shared module. Scan the environment files,
because a tracked `.env` file is the most common source.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `server-only` | Every module that reads a secret, a session, or the backend. | Ships with Next.js | Current | Vercel, active | None |
| Recommend | `gitleaks` | The pre-commit hook and the CI job. `references/dependencies-and-git-workflow.md` owns the command. | Current | Current | Feature complete, security patches only | None |
| Recommend | `pnpm audit --audit-level=high` | A CI gate on every install. Triage each report by severity and by reachability. | pnpm 10.x | Current | Active | None |
| Conditional | `trufflehog` | A second scanner where the team wants a different detection set. | Current | Current | Active | None |
| Conditional | `experimental_taintObjectReference` and `experimental_taintUniqueValue` | Only as a second layer. Both are experimental, and both need `experimental.taint: true`. | react 19.2.x | Current | Meta, active | None |
| Conditional | An `integrity` attribute on a vendor script | Only where the vendor publishes a fixed file. Prefer self-hosting. | Web platform | — | — | — |
| Reject | A credential, a token, or a private address in a `NEXT_PUBLIC_` variable | The build inlines it into a chunk that anybody downloads. | — | — | — | — |
| Reject | A whole configuration record passed as a prop | Serialization carries every field, and not only the rendered ones. | — | — | — | — |
| Reject | A React version read from `package.json` as proof that the application is patched | Next.js bundles its own React Server Components runtime. Read the Next.js version too. | — | — | — | — |
| Reject | An audit report dismissed with no written judgment | It is the plain form of a supply chain failure. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A platform alert reports a leaked key | A `NEXT_PUBLIC_` variable held a secret | Grep the source and scan the built chunks | Rotate the credential, then move the value behind the server boundary |
| A secret is in a chunk and not in `src/` | A barrel file or a shared configuration module carried it | Scan the built output | Split the module, and import `server-only` on the server half |
| A value reaches the browser that no component renders | A whole record crossed as a prop | Read the payload of the route | Pass the fields that the component renders |
| A build fails after a boundary change | A client module imported a `server-only` module | Read the build error, which names the import chain | Move the read to the server, and pass a field |
| An application is patched on `react` and still vulnerable | The Next.js version carries the affected runtime | Read both versions from the installed tree | Upgrade Next.js to the current security release |
| A vendor script changes behavior with no deploy | The vendor rewrote the file | The `integrity` attribute fails, or nothing does | Self-host the file, or pin it by hash |
| An audit reports the same advisory each week | Nobody recorded a judgment | Read the CI log | Triage by severity and reachability, and record the outcome |
| A rotated secret still works | The history rewrite ran, and the rotation did not | Test the old credential | Rotate first, always |

### Version discipline

Read the installed versions before you write code.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| CVE-2025-29927 of March 2025 skips the authorization file, at a CVSS score of 9.1 | A Next.js version from 11.1.4 to 15.2.2 | Upgrade to 15.2.3, 14.2.25, 13.5.9, or 12.3.5, and never authorize in `proxy.ts` alone |
| CVE-2025-55182 of December 2025 is unauthenticated remote code execution in a React Server Components payload, at a CVSS score of 10.0 | React 19.0, 19.1.0, 19.1.1, or 19.2.0 | Upgrade. Read the Next.js version too, because it bundles the affected runtime |
| The December 2025 follow-ups added source exposure and denial of service, and the interim patches were incomplete | React 19.2.2 or 19.2.3 in `package.json` | Move to the current floor of the line, which `references/state-and-effects.md` states |
| The July 2026 security release fixed nine defects | A Next.js version from 16.0.0 to 16.2.10 | Upgrade to 16.2.11 on the 16 line, or to 15.5.21 on the 15 line |
| OWASP Top 10:2025 became final in January 2026, and it adds software supply chain failures | A dependency policy that names no triage step | Add the audit gate, and record a judgment for each report |

## Verification

```bash
# 1. Read the two versions from the installed tree, and never from memory.
node -p "require('next/package.json').version"
npm ls react react-dom

# 2. Find every public variable. Review each hit as a public value.
rg -n 'NEXT_PUBLIC_' src/ .env*

# 3. Find a public variable that names a credential. This prints nothing.
rg -n 'NEXT_PUBLIC_[A-Z_]*(KEY|SECRET|TOKEN|PASSWORD|CREDENTIAL)' src/ .env*

# 4. Confirm that every secret-reading module is server-only. Each file printed
#    here is not.
rg --files-without-match 'server-only' src/lib/dal src/lib/server

# 5. Find a whole record passed as a prop. Read every hit.
rg -n 'config=\{|options=\{|env=\{' -g '*.tsx' src/

# 6. Build, then scan the built chunks for a known secret value.
pnpm build
rg -n "$KNOWN_SECRET_PREFIX" .next/static

# 7. Run the secret scanner over the history.
gitleaks git --redact --verbose

# 8. Run the dependency audit, and fail above the stated severity.
pnpm audit --audit-level=high

# 9. Confirm that no environment file is tracked. This prints nothing.
git ls-files | rg '^\.env'

# 10. Read the payload of a rendered route, and confirm that it holds only the
#     fields that the components render.

# 11. Read the vendor scripts of the deployed application. Each external file
#     carries an integrity attribute, or a written record of why it cannot.
```

## Review checklist

- [ ] Does every `NEXT_PUBLIC_` variable hold a value that is safe in public?
- [ ] Is every credential, token, and private address outside that prefix?
- [ ] Does every module that reads a secret import `server-only`?
- [ ] Does every prop that crosses the boundary carry fields rather than a whole
      record?
- [ ] Was the payload of a rendered route read, and does it hold no private
      value?
- [ ] Do the taint APIs stand as a second layer, and never as the only one?
- [ ] Is the installed Next.js version at or above the current security release?
- [ ] Is the installed React version at or above 19.2.6?
- [ ] Were both versions read from the installed tree, rather than inferred?
- [ ] Does CI run a dependency audit, and does it fail above a stated severity?
- [ ] Does every audit exception carry a written judgment of severity and
      reachability?
- [ ] Does a secret scanner run in the pre-commit hook and in CI?
- [ ] Is the built output scanned, and not the source alone?
- [ ] Is every vendor script self-hosted, or pinned with `integrity` and
      `crossorigin`?
- [ ] Was every leaked credential rotated before the history was rewritten?

## Handoffs

- The `NEXT_PUBLIC_` prefix, `next.config.ts`, and the monthly security release
  → `references/app-router-structure.md`.
- The `server-only` and `client-only` guards, and the import that carries a
  server module into the client bundle →
  `references/server-and-client-components.md`.
- The React security floor, and the advisory family behind it →
  `references/state-and-effects.md`.
- `minimumReleaseAge`, the lifecycle script allowlist, the update bot, the
  lefthook hooks, and the `gitleaks git` command →
  `references/dependencies-and-git-workflow.md`.
- The strategy of a vendor script, its measured cost, its named owner, and the
  facade in front of an embed →
  `references/client-bundle-and-third-party-scripts.md`.
- The policy that scopes a vendor script, and the header set →
  `references/security-headers-and-csp.md`.
- The payload that an endpoint returns, and the destination that it reaches →
  `references/exposed-endpoints-and-destinations.md`.
- The sink that renders a value from a vendor →
  `references/untrusted-markup-and-injection.md`.
- The token, the refresh token, and the store that must never hold either →
  `references/session-and-token-lifecycle.md`.
- The environment variable that a generated client reads →
  `references/openapi-schema-and-codegen.md`.
- The consent that must arrive before a tag manager renders, and the cookie
  inventory → `references/consent-gate-and-cookie-inventory.md`.
- The Docker image, and the build context that produces it →
  `references/build-output-and-container-image.md`. The environment file on the
  host → `references/runtime-process-and-reverse-proxy.md`. The credential
  that a pipeline holds → `references/release-pipeline-and-rollback.md`.
- Secret storage on the server, password hashing, and the Django settings that
  hold a credential → the backend's security review. This file owns what the browser
  receives.
