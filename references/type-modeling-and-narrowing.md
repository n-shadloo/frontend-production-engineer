# Type modeling and narrowing

TypeScript 5.9 baseline, React 19.2.1 or later, `@types/react` 19. This file
owns the vocabulary that models a value inside the program. The subjects are
unions, brands, generics, inference, the rules for a cast, and the component
types that follow from them.

The compiler flags that enforce it are
`references/typescript-config-and-enforcement.md`. The proof that an external
value has the shape it claims is
`references/boundary-validation-and-api-types.md`.

## Principle

A type that can represent a state the program cannot reach is a bug that waits
for input. Model the states that exist, and no others.

Two values of the same primitive type with different meanings are two types.
The compiler cannot tell a user id from an order id while both are `string`.

Inference is more accurate than an annotation, because it keeps the literal.
Constrain a value, and do not widen it.

A union is finished when the compiler fails on a new member. Until then, a new
member is a silent gap.

`unknown` is honest and `any` is not. `unknown` forces the next line to prove
something. `any` deletes every check that follows it.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### `unknown`, never `any`

```ts
// Wrong: any defeats the whole point.
// Failure: every property access is unchecked. payload.userId compiles when
// the field is absent, when it is a number, and when payload is null.
function handle(payload: Record<string, any>) {
  return payload.userId;
}
```

```ts
// Correct: accept unknown, and prove the shape before you read it.
import { z } from "zod";

function handle(payload: unknown) {
  return z.object({ userId: z.string() }).parse(payload).userId;
}
```

A package with wrong types earns a module augmentation or a small `.d.ts`
shim, plus a typed wrapper. It never earns a cast at the call site.

```ts
// types/legacy-charts.d.ts
declare module "legacy-charts" {
  export function render(node: HTMLElement, series: number[]): void;
}
```

### Make the illegal state unrepresentable

```ts
// Wrong: parallel optional fields.
// Failure: the type permits { isLoading: true, error: Error, data: Order },
// which the program never produces. Every reader writes a different guess
// about which field wins, and the guesses disagree.
type OrderState = {
  isLoading?: boolean;
  error?: Error;
  data?: Order;
};
```

```ts
// Correct: one discriminant, three reachable states.
type OrderState =
  | { status: "loading" }
  | { status: "error"; error: Error }
  | { status: "ok"; data: Order };
```

The same rule applies to a model of a backend resource. A serializer with
every field optional hides which fields the contract guarantees, and it pushes
an `undefined` test into every reader. Mark `?` only on a field the contract
permits to be absent.

```ts
// Wrong: optional everything, "to be safe".
// Failure: every reader tests for undefined on a field that the contract
// always sends, and no reader can tell which fields the contract guarantees.
type Order = { id?: string; total?: number; status?: string };
```

```ts
// Correct: the guarantees are exact, and the states are a union.
type Order =
  | { status: "pending"; id: OrderId; total: number }
  | { status: "shipped"; id: OrderId; total: number; trackingUrl: string };
```

Which fields the DRF contract guarantees, and how a serializer expresses that,
is `references/boundary-validation-and-api-types.md`.

### Exhaustiveness is enforced

```ts
// Wrong: the switch handles the variants that exist today.
// Failure: a fourth status is added later, this compiles, and label()
// returns undefined for it. The gap surfaces as an empty cell in the UI.
function label(s: OrderState) {
  switch (s.status) {
    case "loading": return "…";
    case "ok": return "OK";
    case "error": return "Error";
  }
}
```

```ts
// Correct: assertNever turns a new variant into a compile error.
export function assertNever(x: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`);
}

function label(s: OrderState): string {
  switch (s.status) {
    case "loading": return "…";
    case "ok": return "OK";
    case "error": return "Error";
    default: return assertNever(s);
  }
}
```

Every union `switch` ends in a `default` that calls `assertNever`. The lint
rule `switch-exhaustiveness-check` reports the ones that do not.

### Brand an identifier

Give a brand to every id, slug, token, and money amount.

```ts
// Wrong: both parameters are string, so the call site can swap them.
// Failure: cancelOrder(orderId, userId) compiles. The bug reaches production
// and reads as a permissions failure.
function cancelOrder(userId: string, orderId: string) {}
```

```ts
// Correct: the brand makes the swap a compile error.
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

function cancelOrder(userId: UserId, orderId: OrderId) {}
```

Where a schema already describes the value, brand it in the schema instead.
`.brand<"UserId">()` produces `string & z.$brand<"UserId">`, which is static
only and adds no run-time cost. Use the phantom-property form above only where
no schema exists.

### `satisfies` keeps the inference and adds the check

```ts
// Wrong: the annotation widens the value.
// Failure: metadata.title has type string | TemplateString | null, so no
// reader can depend on the literal "Home".
import type { Metadata } from "next";
export const metadata: Metadata = { title: "Home" };
```

```ts
// Correct: satisfies checks the shape and keeps the literal.
import type { Metadata } from "next";
export const metadata = { title: "Home" } satisfies Metadata;
```

Order matters when both operators appear. Write `as const satisfies T`. The
reverse, `satisfies T as const`, is an error.

### A const object instead of an `enum`

```ts
// Wrong: a runtime enum.
// Failure: it emits JavaScript, a numeric enum accepts any number, and the
// declaration breaks a transpiler that erases types only.
enum Role { Admin, User }
```

```ts
// Correct: a const object and a derived union.
const Role = { Admin: "admin", User: "user" } as const;
type Role = (typeof Role)[keyof typeof Role]; // "admin" | "user"
```

`enum` is current but in decline. Use `enum` only where an external interface
demands the runtime object. Audit the ones already present; add none.

### `type` by default, `interface` to merge

Use `type` for every declaration. Use `interface` only where another module
must merge into the declaration, which is the module augmentation case above.
One house rule removes a whole class of review comment.

### The absent value has one model

Use `undefined` for "the program did not provide it". Use `null` only where
the backend sends JSON `null`. An optional property means "the key may be
absent", never "the value may be null". A type that mixes the two makes every
reader test for both.

### Indexed access returns `T | undefined`

```ts
// Wrong: the non-null assertion silences the check.
// Failure: getElementById returns null when the element is absent, and the !
// converts a clear compile error into a run-time crash on appendChild.
const el = document.getElementById("root")!;
el.appendChild(node);
```

```ts
// Correct: test for the absence, and handle it.
const el = document.getElementById("root");
if (!el) throw new Error("#root missing");
el.appendChild(node);
```

`noUncheckedIndexedAccess` gives `arr[i]` and `record[key]` the type
`T | undefined`. Guard the value, supply a default with `??`, or read through
a checked accessor. NEVER reach for `!`.

### The ladder before a cast

A cast is the last rung, not the first. Work down the ladder in order.

| Question | If yes | It reverses when | The cost |
| --- | --- | --- | --- |
| Does the value come from outside the program? | Parse it. `references/boundary-validation-and-api-types.md` | Never. Nothing else proves a value from outside the program. | A schema for each boundary, and a parse on each value that crosses it. |
| Does a package ship a wrong or a missing type? | Write a module augmentation or a `.d.ts` shim, and a typed wrapper. | The package ships correct types, so the shim then hides them. | A declaration that a package upgrade can make wrong with no report. |
| Can a test prove the narrow? | Write a type predicate: `function isOrder(v: unknown): v is Order`. | The value comes from outside the program, so the first row applies. | The compiler trusts the predicate, so a wrong body gives an unsound narrow. |
| Is it a literal that must not widen? | `as const`. | The value must be changed after it is declared. | Every property becomes readonly, and a mutable copy needs a spread. |
| Is it a narrow that the compiler cannot see? | `as`, with a comment that proves the soundness. | A predicate can prove the narrow, which the third row covers. | The comment is the only proof, and no check reports it when the code changes. |

Every remaining `as` in the codebase is a const assertion, or it carries that
comment. There is no third case.

### A fallible call returns a result

Return a discriminated union where a failure is expected rather than
exceptional. Throw only for the exceptional case, which the segment
`error.tsx` then renders.

```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
```

Use a function overload only where the return type genuinely differs per
argument shape. Prefer one union parameter everywhere else.

### Component types

`React.FC` is current but in decline. `forwardRef`, `MutableRefObject`, and the
global `JSX` namespace are alive only in legacy code.

```tsx
// Wrong: React.FC.
// Failure: it weakens a generic component, it carries history that this
// codebase does not want, and react.dev discourages it.
const Card: React.FC<{ title: string }> = ({ title }) => <h2>{title}</h2>;
```

```tsx
// Correct: a plain function, with children declared where children exist.
import type { PropsWithChildren } from "react";

function Card({ title, children }: PropsWithChildren<{ title: string }>) {
  return <section><h2>{title}</h2>{children}</section>;
}
```

```tsx
// Wrong: forwardRef on a new component.
// Failure: React 19 makes ref a normal prop. The two type parameters add
// complexity and give no benefit. react.dev states that forwardRef is no
// longer necessary.
import { forwardRef } from "react";
const Input = forwardRef<HTMLInputElement, { label: string }>((props, ref) => (
  <input ref={ref} aria-label={props.label} />
));
```

```tsx
// Correct: ref is a typed prop.
import type { Ref } from "react";

function Input({ label, ref }: { label: string; ref?: Ref<HTMLInputElement> }) {
  return <input ref={ref} aria-label={label} />;
}
```

```tsx
// Wrong: the context is read through a non-null assertion.
// Failure: the ! lies whenever a consumer renders outside the Provider. The
// error surfaces as a property read on null, far from the cause.
const Ctx = createContext<AuthState | null>(null);
const useAuth = () => useContext(Ctx)!;
```

```tsx
// Correct: the hook throws once, with a message that names the fix.
import { createContext, useContext } from "react";

const Ctx = createContext<AuthState | null>(null);

export function useAuth(): AuthState {
  const value = useContext(Ctx);
  if (value === null) throw new Error("useAuth must be used within <AuthProvider>");
  return value;
}
```

`@types/react` 19 changes four more things. `useRef` requires an initial
argument. `MutableRefObject` is deprecated, and `RefObject<T>` now means
`RefObject<T | null>`. `ReactElement.props` has type `unknown` rather than
`any`, so `isValidElement` and `cloneElement` need an explicit type argument.
The global `JSX` namespace is deprecated; use `React.JSX`.

Take the props of an existing element or component with
`ComponentProps<"button">` or `ComponentProps<typeof Card>`. Do not restate
them.

## Verification

```bash
# 1. Find every `any`. Each hit is a finding until it carries a reason.
rg -nw 'any' src/

# 2. Find a non-null assertion. Read every hit.
rg -n '!\.\w|!\)|!;' src/

# 3. Find a cast. Each hit is a const assertion, or it carries a comment.
rg -n ' as [A-Z]' src/

# 4. The lint gate reports a union switch with no exhaustive default, and
#    every unsafe read of an `any`.
pnpm exec eslint .

# 5. Find the deprecated component idioms. This must print nothing.
rg -n 'React\.FC|forwardRef|MutableRefObject' src/

# 6. Find a runtime enum. Audit every hit.
rg -n '^\s*(export )?enum ' src/

# 7. Find an interface that no module augments.
rg -n '^\s*(export )?interface ' src/
```

## Review checklist

- [ ] Is `any` absent, apart from a line that carries a reason?
- [ ] Does every value from a package with weak types pass through a module
      augmentation or a typed wrapper?
- [ ] Does every multi-state value use a discriminated union rather than
      parallel optional fields?
- [ ] Does every union `switch` end in a `default` that calls `assertNever`?
- [ ] Do the ids, the tokens, and the money amounts carry a brand?
- [ ] Does every config object use `satisfies` rather than an annotation?
- [ ] Is `as const satisfies T` written in that order?
- [ ] Is a const object with a derived union used in place of `enum`?
- [ ] Is `type` the default, with `interface` only where a merge is intended?
- [ ] Is `undefined` used for absence and `null` only for a JSON `null`?
- [ ] Is every non-null `!` absent from a value that can be absent?
- [ ] Is every remaining `as` a const assertion, or does it carry a comment
      that proves the narrow?
- [ ] Is every component a plain function, with no `React.FC`?
- [ ] Does every new component take `ref` as a prop rather than through
      `forwardRef`?
- [ ] Does every context hook throw on a missing Provider rather than assert?

## Handoffs

- The compiler flags that report these failures, and the lint rules that
  enforce them → `references/typescript-config-and-enforcement.md`.
- The schema for a value that enters from outside the program →
  `references/boundary-validation-and-api-types.md`.
- The `"use client"` boundary that a component sits on, and what a prop must
  serialize to cross it → `references/server-and-client-components.md`.
- Composition, the slots, the controlled and uncontrolled choice, and the list
  key → `references/component-composition.md`.
- Where the state of a component lives, the effect rules, and the Rules of
  React → `references/state-and-effects.md`.
- The React 19 Actions, and the boundary that renders while a value is absent →
  `references/suspense-and-actions.md`.
- The theme object and the token types behind it → domain 09
  `design-system-and-styling`. Not integrated yet. The `satisfies` rule above
  covers only the type concern.
- The form state type and the field-level error map → domain 11
  `forms-and-validation`. Not integrated yet.
