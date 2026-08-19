# OpenAPI schema and codegen

drf-spectacular 0.30.0, DRF 3.17, OpenAPI 3.0.3, openapi-typescript 7.13. This
file owns the OpenAPI schema as the one source of the frontend types. It also
owns the generator that reads the schema, and the gate that proves the output.

The client that calls an endpoint is
`references/api-client-and-request-safety.md`. The types that a response
produces, and the parse beside them, are
`references/boundary-validation-and-api-types.md`. The script name and the
output folder are `references/directory-and-module-boundaries.md`.

## Principle

A schema is a contract only where one side generates from it. A type that a
person copied is an opinion about the contract, and the two drift apart.

A generated file is an output. The command that produces it is the source, and
the command belongs in the repository.

A contract breaks in silence. Nothing reports a renamed field until a user
reads `undefined`, so a gate must run the generator and then the compiler.

A build that reads a live URL reads a target that changes. Pin an artifact, and
the build repeats.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### What the backend publishes

`drf-yasg` and its Swagger 2.0 output are alive only in legacy code.

| The backend runs | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| DRF with drf-spectacular | Consume the document. Require `COMPONENT_SPLIT_REQUEST: True`. | The backend team refuses the setting, so the request and the response share one component. | Two components for each endpoint, so the generated file is larger and each read names the correct one. |
| DRF with `drf-yasg` | The document is Swagger 2.0. Convert it to OpenAPI 3.0 in the pipeline, or ask for a move to drf-spectacular. | The backend moves to drf-spectacular, and the first row then applies. | A conversion step in the pipeline, and every conversion defect reaches the types. |
| django-ninja | The document is OpenAPI 3.1.0. Take a generator that reads 3.1. | The chosen generator drops 3.1 support. | The generator choice is narrower than the table below states. |
| A schema that a person wrote | Treat it as untrusted. Lint it, and parse every response that it describes. | The backend starts to generate the document. | A parse at every boundary that the document describes. |
| No schema, or a schema that is provably wrong | STOP, and open the task with the backend team. | Never, while no correct document exists. | The frontend task waits for the backend team. |

Where no schema exists, write a Zod schema at the one boundary that the current
task needs. Mark it `// TEMP: no schema — remove when the schema lands`. NEVER
grow that schema into a second model of the whole API.

drf-spectacular 0.30.0 emits OpenAPI 3.0.3, 3.1, and 3.2. The `OAS_VERSION`
default stays `3.0.3`, and that version is the best tested. Ask the backend
team to validate the schema in CI, with `--validate --fail-on-warn`. A document
with warnings produces types with holes.

### `COMPONENT_SPLIT_REQUEST` is not optional here

The upstream default is `False`, which gives one component for the request and
the response of an endpoint. The frontend needs `True`.

The split gives two components. `X` describes what the endpoint returns, and
`XRequest` describes what it accepts. A `FileField` is a URL string in the
response and a file in the request, so one component cannot describe both.
`COMPONENT_SPLIT_PATCH` defaults to `True`, and it gives the partial component
that a PATCH accepts.

CAUTION: the split has a cost that upstream issue #810 records. A response
field becomes optional in the response component when it is nullable, when it
has a default, or when it is not `readOnly`. The generated type then carries
`?`, and every read needs an optional chain. Ask the backend team for an
`@extend_schema_field` annotation, or for an explicit `required` list, on the
fields that matter. Until that lands, read the field through the optional
chain that `noUncheckedIndexedAccess` already forces.

### How the schema reaches the frontend

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| One CI pipeline, or one repository, holds both sides | Commit `schema.yml`, and let `api:generate` read the committed file. | The two sides split into two repositories, which the next row covers. | Every contract change adds a schema diff to the pull request. |
| Two repositories | Django CI publishes a versioned `schema.yml` artifact. The frontend pins the URL and the hash. | The two repositories merge, and the row above then applies. | An artifact store, a publish step on the backend, and a pinned hash that a human must move. |
| A developer machine | The live path `/api/schema/` is acceptable. Commit the snapshot that CI reads. | The command runs in CI or in a build, where a live URL makes the build unrepeatable. | The types follow whatever the local server serves, so two machines can disagree. |

```jsonc
// Wrong: package.json generates from a live URL.
// Failure: the build depends on a server that is up, reachable, and current.
// Two builds of the same commit produce two clients. The types pass in
// development against the development server, and the deployed API sends a
// different shape.
{ "scripts": { "api:generate": "openapi-typescript $DJANGO_URL/api/schema/ -o src/api/generated/schema.d.ts" } }
```

```jsonc
// Correct: the generator reads a file that the repository or the artifact
// store holds.
{ "scripts": { "api:generate": "openapi-typescript schema.yml -o src/api/generated/schema.d.ts" } }
```

`references/lint-format-and-scripts.md` owns the script surface that holds this
command. `references/directory-and-module-boundaries.md` owns the output path,
and it states why the folder stays out of version control.

### Choose the generator

`openapi-generator` and `swagger-typescript-api` are alive only in legacy code.

| Condition | Choice | It reverses when | The cost |
| --- | --- | --- | --- |
| The default for this stack | `openapi-typescript` for the types, and `openapi-fetch` for the runtime | The project needs generated hooks or handlers, which the next row covers. | A person writes each query hook and each mock. |
| The project wants generated TanStack Query hooks, MSW handlers, and one file for each tag | Orval | The Node floor stays at 20.9, because Orval needs Node 22.18 or later. | A large generated surface, and a config that decides the shape of application code. |
| The project wants an SDK and many plugins from one config | `@hey-api/openapi-ts` | The package reaches 1.0 with a stable config, or the project cannot accept a below-1.0 pin. | The package is below 1.0, so the exact version is pinned and each bump is read. |
| The schema is very large, and the output must tree-shake for each tag | The scoped `@kubb/*` packages | The schema is small enough that one file costs nothing. | Several scoped packages to pin, and a version report that the plain package does not confirm. |
| `openapi-generator` is already installed and working | Audit only. Add no new install of it. | Never. The next row of the table below states the reason. | A Java toolchain in the build, and a 401 that does not reach `onError`. |
| `swagger-typescript-api` is already installed | Audit only. Plan the move. | Never. The move goes to the first row of this table. | The move rewrites every call site that the generated client serves. |

`openapi-typescript` emits types and no runtime. `openapi-fetch` is the
companion at about 6 kB, it calls the platform `fetch`, and it assumes no
browser global. It therefore runs in a Server Component, a Route Handler, and
the browser without a second code path.

Three conditions rule out a choice on this stack. Orval needs Node 22.18 or
later, and that floor is above the Node 20.9 floor of Next 16. Raise the floor
first, or take another generator. `@hey-api/openapi-ts` is below 1.0, so pin
the exact version. The plain `kubb` package reports a version that the scoped
`@kubb/*` packages do not confirm. Install the scoped packages, and pin them.

NEVER introduce `openapi-generator`. It needs a Java toolchain, and its
`typescript-fetch` output does not reach `onError` on a 401, which issue #17979
records.

### The drift gate

```bash
# Wrong: CI generates the client and proves nothing.
pnpm api:generate
```

```bash
# Correct: CI generates, then compiles, then compares the two schemas.
pnpm api:generate && pnpm typecheck
oasdiff breaking schema.base.yaml schema.head.yaml --fail-on ERR
```

The compiler is the gate. A serializer field that the backend renamed becomes a
compile error on the first line that reads it, under `strict` and
`noUncheckedIndexedAccess`. The generated folder is out of version control, so
a `git diff` on it always passes and proves nothing.

`oasdiff` compares the committed schema against the schema of the backend
change. It exits non-zero on a breaking change, and it reports an enum value
that the union does not carry. Run it as a gate, never as a report.

### What the generator gets wrong

| Symptom in the generated types | Cause | Action |
| --- | --- | --- |
| A field has type `any` | A `SerializerMethodField` with no annotation | Ask for `@extend_schema_field` on the backend. NEVER cast the field. |
| A response field is optional and should not be | The `COMPONENT_SPLIT_REQUEST` behavior of issue #810 | Read it through the optional chain, and ask for an explicit `required`. |
| A field is not nullable and the API sends `null` | `nullable: true` beside a `$ref`, which OpenAPI 3.0.3 ignores | Add `.nullable()` in the boundary schema. `references/boundary-validation-and-api-types.md` owns the detail. |
| An enum is a TypeScript `enum` | The generator default | Take the const union. In Orval the key is `enumGenerationType: 'const'`. |
| A nullable field became `field?: T` | An older generator that treats `nullable` as optional | Regenerate with the pinned version. Nullable is not optional. |
| An enum name is unreadable, or two enums collide | No `ENUM_NAME_OVERRIDES` on the backend | Name the enums that the frontend reads, and ask for the override. |

```ts
// Wrong: the any from an unannotated SerializerMethodField gets a cast.
// Failure: the cast asserts a shape that the schema never declared, so the
// generator keeps emitting any and nothing ever reports the gap. The next
// backend change to that method ships as a run-time crash.
const total = (order as { total_display: string }).total_display;
```

```ts
// Correct: the value stays unknown until a schema proves it, and the
// annotation is the real fix.
// TODO: backend to add @extend_schema_field on total_display (FE-233).
const total = z.string().parse(order.total_display);
```

### The case convention is decided once

| Condition | Action | It reverses when | The cost |
| --- | --- | --- | --- |
| The team wants camelCase in TypeScript | Convert at the client boundary with `humps` or `ts-case-convert`, OR take `djangorestframework-camel-case` on the backend with the drf-spectacular `camelize_serializer_fields` hook. | The backend team refuses the hook, and the run-time conversion fails the payload budget. | A conversion on every response, or a backend dependency that every serializer passes through. |
| The backend is snake_case, and the team is small | Keep snake_case from end to end. The generated types carry it, and nothing converts. | The team grows, or a style rule rejects a snake_case identifier in TypeScript. | The TypeScript code holds two conventions, one for the API and one for the application. |
| Some files convert, and some do not | FORBIDDEN. Take one convention, and convert at one place. | Never. Two conventions in one codebase produce a field that no reader can name. | The move to one convention touches every file that reads a response. |

The backend option gives camelCase in the schema, so the generated types are
camelCase and no run-time conversion exists. The frontend option leaves the
schema in snake_case. The conversion then sits at the client boundary, and the
generated types do not describe what the code reads. Take the backend option
where the backend team agrees. Record the choice, because a reader cannot
derive it from one file.

### The error envelope decides the normalizer

Default DRF sends `{"field": ["msg"]}` on a 400, and `{"detail": "..."}` on a
401, a 403, a 404, a 405, and a 429. `drf-standardized-errors` sends an
envelope with a `type` field and an `errors` array instead.

Read which one the backend sends before you write the map.
`references/api-client-and-request-safety.md` owns `normalizeApiError`, and
`references/boundary-validation-and-api-types.md` owns the type of each
envelope.

## Verification

```bash
# 1. Confirm that the generator reads a file and not a live URL.
rg -n '"api:generate"' package.json

# 2. Confirm the split-request setting in the schema. Both must appear.
rg -n 'Request:$|Request:' schema.yml | head

# 3. Generate, then compile. An error is drift.
pnpm api:generate && pnpm typecheck

# 4. Gate the backend change against the committed schema.
oasdiff breaking schema.base.yaml schema.head.yaml --fail-on ERR

# 5. Find an any in the generated types. Each hit needs an annotation on the
#    backend.
rg -n ': any' src/api/generated/

# 6. Find a TypeScript enum in the generated types. This must print nothing.
rg -n '^\s*enum ' src/api/generated/

# 7. Find a hand-written response model beside the generated one.
rg -n 'interface .*(Response|Payload|Dto)\b' src/ -g '!src/api/generated/**'

# 8. Confirm one case convention. Snake hits with no package is snake end to
#    end. No snake hits with no package is the backend hook. Snake hits with a
#    package is the client boundary. Any other pair is two conventions.
rg -n '[a-z]_[a-z].*:' src/api/generated/schema.d.ts | head
rg -n 'camelcase|humps|ts-case-convert' package.json
```

## Review checklist

- [ ] Does the frontend generate every response type from the schema?
- [ ] Does `api:generate` read a committed or pinned artifact, never a live
      URL?
- [ ] Is `COMPONENT_SPLIT_REQUEST` set to `True` on the backend?
- [ ] Does the code consume `XRequest` for a request and `X` for a response?
- [ ] Does CI run `api:generate` and then the typecheck?
- [ ] Does a breaking-change gate compare the two schemas?
- [ ] Is the generator one of the recommended choices, at a pinned version?
- [ ] Does the Node floor allow the generator that the project installed?
- [ ] Is every `any` in the generated types raised with the backend rather than
      cast away?
- [ ] Is every enum a const union rather than a TypeScript `enum`?
- [ ] Is the case convention recorded, and does one place convert?
- [ ] Is the error envelope of the backend established before the normalizer is
      written?
- [ ] Is a temporary hand-written schema marked, scoped to one boundary, and
      tracked?

## Handoffs

- The one client, the base URLs, the retry rule, and `normalizeApiError` →
  `references/api-client-and-request-safety.md`.
- CORS, CSRF, the cookie, and the proxy Route Handler →
  `references/cross-origin-and-bff-proxy.md`.
- The types that a DRF construct produces, the pagination envelope type, the
  error envelope types, and the parse →
  `references/boundary-validation-and-api-types.md`.
- The output folder, the `.gitignore` entry, and the rule that no file inside
  it is a source file → `references/directory-and-module-boundaries.md`.
- The `package.json` script surface that holds `api:generate` →
  `references/lint-format-and-scripts.md`.
- The module that calls the backend, and the Server Action that mutates →
  `references/data-access-and-mutations.md`.
- The serializer, the viewset, the status code, and the deprecation of a field →
  the backend. The backend owns the server side of this contract, and this file
  owns what the frontend generates from it.
- The query keys and the cache built on the generated types →
  `references/server-state-and-query-cache.md`.
- The MSW handlers and the contract test against the schema →
  `references/network-mocks-and-contract-tests.md`.
- The CI job that downloads the schema artifact, and the order of the two
  deploys that change the contract →
  `references/release-pipeline-and-rollback.md`.
