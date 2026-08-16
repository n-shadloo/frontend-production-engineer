# Boundary validation and API types

TypeScript 5.9 baseline, Zod 4, against a Django and DRF backend that
publishes OpenAPI 3.0.3. This file owns every value that enters the program
from outside it, and the types that a DRF response produces on the frontend.
The vocabulary that models the value once it is inside is
`references/type-modeling-and-narrowing.md`. The compiler flags and the lint
gate are `references/typescript-config-and-enforcement.md`. Where the call to
the backend belongs is `references/data-access-and-mutations.md`.

## Principle

A type on a value from outside the program is a claim. The compiler checks no
claim about the world. It checks only that the rest of your code agrees with
the claim.

The place to check is the place the value enters. One check at the edge types
every line downstream of it.

A generated type describes what the server promised. A parse proves what the
server sent. The two are different facts, and a contract drifts between them.

A cast on unparsed data is the most common failure in this domain. It looks
like a type and behaves like a comment.

## Pinned-stack depth

### The boundaries

Every one of these is outside the program. Each needs a parse.

| Boundary | The value |
| --- | --- |
| The network | A `fetch` response, a WebSocket message, a Server-Sent Event |
| The URL | `searchParams`, a route `param`, a hash fragment |
| Storage | `localStorage`, `sessionStorage`, `IndexedDB`, a cookie |
| The environment | `process.env` |
| The user | `FormData`, a file, a pasted string |
| A third party | A webhook body, an OAuth callback, an analytics payload |

### Parse the response, never cast it

```ts
// Wrong: the cast is a lie.
// Failure: res.json() has type any, so the cast asserts a shape that nothing
// checked. A renamed DRF serializer field ships as a run-time crash on the
// first property read, in production, with no compile error anywhere.
const res = await fetch("/api/users/1");
const user = (await res.json()) as User;
```

```ts
// Correct: the type is proven from the validator.
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().brand<"UserId">(),
  email: z.email(),
  displayName: z.string(),
});
type User = z.infer<typeof UserSchema>;

export async function getUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const parsed = UserSchema.safeParse(await res.json());
  if (!parsed.success) throw new Error(z.prettifyError(parsed.error));
  return parsed.data;
}
```

`safeParse` returns a result rather than a throw, so the caller decides. Take
the static type from `z.infer`. Use `z.input` and `z.output` where a
transform, a `.default()`, or a coercion makes the two differ.

### The generated type and the schema divide the work

| Condition | Action |
| --- | --- |
| The DRF schema is generated and the endpoint is not trust-sensitive | Take the shape from the generated `paths` and `components` types. Add no schema. |
| The endpoint carries authentication, money, or input echoed to another system | Take the shape from the generated types, and prove the value with a thin schema as well. |
| The call is a hand-written `fetch` | Parse. `res.json()` has type `any`, so nothing else proves the value. |
| No generated schema exists yet | Parse, and open the codegen task. A hand-written schema is the temporary state, never the target. |

NEVER hand-copy the OpenAPI model into TypeScript or into Zod for every
endpoint. Two sources for one shape drift in silence, and the drift surfaces
as a run-time crash. Domain 05 `django-drf-api-contract` owns the generation
config and the codegen command. The sibling skill `django-api-contract` owns
the server side of the contract.

### What a DRF construct produces

| DRF construct | The OpenAPI 3.0.3 emission | The correct TypeScript | What breaks on drift |
| --- | --- | --- | --- |
| `required=False` | The field is absent from `required` | `field?: T` | A required model crashes on `undefined`. |
| `allow_null=True` | `nullable: true` | `field: T \| null` | See the note below. The generated type is often not nullable. |
| `required=False, allow_null=True` | Absent from `required`, and `nullable: true` | `field?: T \| null` | A collapse of the two loses PATCH semantics: an absent key means "no change", and `null` means "clear". |
| `blank=True` on a `CharField` | The field accepts `""` | `field: string` | `value \|\| fallback` discards a legitimate empty string. Test `=== ""`. |
| Default DRF errors | `{ "field": ["msg"], "non_field_errors": ["msg"] }` | `Record<string, string[]>` | A `{ detail }` model misses every field error, so a form maps nothing. |
| `drf-standardized-errors` | `{ "type", "errors": [{ "code", "detail", "attr" }] }` | A discriminated union on `type` | A raw-DRF model breaks all error handling. |
| `PageNumberPagination` | `{ "count", "next", "previous", "results" }` | `Paginated<T>` | A `T[]` model crashes on `.results`. |

```ts
// The pagination envelope, once, as a generic.
export type Paginated<T> = {
  count: number;
  next: string | null;
  previous: string | null;
  results: T[];
};
```

```ts
// The drf-standardized-errors envelope, as a discriminated union.
export type ApiError =
  | { type: "validation_error"; errors: { code: string; detail: string; attr: string }[] }
  | { type: "client_error"; errors: { code: string; detail: string; attr: string | null }[] }
  | { type: "server_error"; errors: { code: string; detail: string; attr: string | null }[] };
```

`attr` names the field. A nested field arrives as `shipping_address.zipcode`,
with a separator the server configures. Split on that separator to reach the
form field. The field-level map into a form is domain 11
`forms-and-validation`.

### The nullable field that the generated type does not carry

OpenAPI 3.0.3 adds `null` to the type that `type` declares, and only where
`type` is declared in the same schema object. drf-spectacular emits
`nullable: true` beside a `$ref` or an `allOf`, where there is no adjacent
`type`. The keyword then does nothing. The generated TypeScript says the field
is never null, and the API sends `null`.

The symptom is a run-time crash on a field that TypeScript called non-null.
Detect it by a comparison of the generated type against a live payload. Fix it
on the frontend with a `.nullable()` in the boundary schema, and raise the
emission with domain 05 `django-drf-api-contract`. The `blank` and `null`
choice setting on the enum generator is the other toggle that domain owns.
Name it and hand it over rather than change it here.

### Zod 4

Zod 4 is the pin. The root `"zod"` export is version 4.

| Task | The call |
| --- | --- |
| Define an object | `z.object({ ... })` |
| Parse without a throw | `.safeParse(value)` |
| Static type from a schema | `z.infer`, and `z.input` or `z.output` where they differ |
| A branded primitive | `.brand<"UserId">()`, which yields `T & z.$brand<"UserId">` |
| One of several shapes | `z.discriminatedUnion("status", [...])` |
| A string format | `z.email()`, `z.uuid()`, `z.ipv4()`, `z.ipv6()`, `z.cidrv4()`, `z.cidrv6()` |
| A message for a person | `z.prettifyError(error)` |
| A tree for a nested form | `z.treeifyError(error)` |
| A flat map for a simple form | `z.flattenError(error)` |
| The list of problems | `error.issues` |
| A cross-field rule | `.superRefine()` |

`.superRefine()` is supported. An early Zod 4 release marked it deprecated,
and the deprecation was reverted. Prefer it for a cross-field rule. Reach for
`.check()`, which is the lower-level and faster call, only on a
performance-critical path.

### The Zod 3 calls that Zod 4 replaced

Each one of these ships clean. The removed calls throw when the module loads,
or read `undefined` at run time. The deprecated calls still run, and the next
major version removes them. Search for all of them after any upgrade.

| Zod 3 | Zod 4 | What happens if it stays |
| --- | --- | --- |
| `error.errors` | `error.issues` | Reads `undefined`. The error handler renders nothing. |
| `.format()`, `.flatten()` | `z.treeifyError()`, `z.flattenError()` | Deprecated. |
| `z.string().ip()`, `.cidr()` | `z.ipv4()`, `z.ipv6()`, `z.cidrv4()`, `z.cidrv6()` | Removed. The application throws when the module loads. |
| `z.string().email()` | `z.email()` | Deprecated. The format calls moved to the top level. |
| `z.nativeEnum()` | `z.enum()` | Deprecated. |
| `.merge()` | `.extend()` | Deprecated. |
| An invalid discriminator | A valid literal discriminant | Throws when the schema is created, not when a value is parsed. |

Three behaviors changed without a rename. `.default()` now applies to the
output type and short-circuits on `undefined`; the old input-side behavior is
`.prefault()`. The input type of `z.coerce.*` is now `unknown`. Error
customization is unified under one `error` parameter.

Zod 4 requires TypeScript 5.5 or later. Reach for `valibot` only where the
client bundle budget rules out Zod. Reach for `arktype` only where a schema
must be written in TypeScript syntax. Measure both against Zod Mini before you
adopt either.

### The other boundaries

```ts
// Wrong: the search parameter is read as a typed value.
// Failure: ?page=abc gives NaN, the query asks Django for page NaN, and the
// route renders an error the user cannot act on.
const { page: raw } = await props.searchParams;
const page = Number(raw);
```

```ts
// Correct: parse the query string, with a default.
const QuerySchema = z.object({
  page: z.coerce.number().int().positive().catch(1),
});
const { page } = QuerySchema.parse(await props.searchParams);
```

Validate the environment once, when the process starts. Reference every
variable explicitly, because Next.js bundles only the variables that the code
names. `@t3-oss/env-nextjs` builds the schema and separates the server
variables from the `NEXT_PUBLIC_` variables. A hand-written Zod schema in a
single `env.ts` module does the same job. Either way the process must fail at
boot, never at the first request.

Treat `localStorage` and a cookie the same way. Another tab, an older release
of the application, and the user all write to them.

## Verification

```bash
# 1. Find a cast on a parsed response. This must print nothing.
rg -n '\.json\(\)\)? as ' src/

# 2. Find a module that reads an external value and never parses one.
rg -l 'fetch\(|localStorage|sessionStorage|process\.env' src/ \
  | xargs rg --files-without-match 'safeParse|\.parse\('

# 3. Find the Zod 3 calls. This must print nothing.
rg -n '\.errors\b|\.format\(\)|\.flatten\(\)|z\.string\(\)\.email\(\)' src/
rg -n 'z\.nativeEnum|\.merge\(|z\.string\(\)\.ip\(|z\.string\(\)\.cidr\(' src/

# 4. Regenerate the types from the DRF schema, then diff. A diff is drift.
git diff --exit-code -- lib/api

# 5. Compare a generated nullable field against a live payload.
curl -s "$DJANGO_URL/api/orders/1/" | jq .

# 6. Confirm that a paginated endpoint is typed as the envelope.
rg -n 'Paginated<|results' lib/api lib/dal

# 7. Confirm that the environment is validated at boot.
rg -n 'createEnv|EnvSchema' src/env.ts
```

## Review checklist

- [ ] Does every value from the network, the URL, storage, the environment, a
      form, or a webhook pass through a schema?
- [ ] Is every static type derived from the parse result rather than from a
      cast?
- [ ] Is `res.json()` never cast to a type?
- [ ] Do the response shapes come from the generated `paths` and `components`
      types rather than a hand-copied model?
- [ ] Does every trust-sensitive endpoint also carry a schema?
- [ ] Is every paginated endpoint typed as the envelope, never as an array?
- [ ] Does the error type match the envelope the server actually sends?
- [ ] Does every field that DRF declares `allow_null=True` carry `| null` on
      the frontend, even when the generated type omits it?
- [ ] Are an absent field and a `null` field kept apart?
- [ ] Is an empty string treated as a value rather than as absence?
- [ ] Is every Zod 3 call replaced by its Zod 4 form?
- [ ] Does the process validate the environment at boot and fail there?
- [ ] Does `z.prettifyError`, `z.treeifyError`, or `z.flattenError` produce
      every message that a person reads?

## Handoffs

- The vocabulary that models the parsed value — a union, a brand, a
  narrow → `references/type-modeling-and-narrowing.md`.
- The compiler flags and the lint rules that report an unparsed value →
  `references/typescript-config-and-enforcement.md`.
- Where the call to the backend belongs, and the data access layer that holds
  it → `references/data-access-and-mutations.md`.
- The drf-spectacular config, the codegen command, the schema artifact, and
  the CSRF and CORS rules → domain 05 `django-drf-api-contract`. Not
  integrated yet. The server side belongs to the sibling skill
  `django-api-contract`.
- The query keys, the cache, and the mutation state built on these types →
  domain 06 `data-fetching-and-state`. Not integrated yet.
- The resolver, the field array, and the map from `attr` to a form field →
  domain 11 `forms-and-validation`. Not integrated yet.
- The words in a validation message that a user reads → domain 15
  `ux-writing-and-content-design`. Not integrated yet.
- The MSW handlers and the contract test against the schema → domain 20
  `testing-and-quality`. Not integrated yet.
