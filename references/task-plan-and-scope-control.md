# Task plan and scope control

Next.js 16.3, React 19.2.6 or later, TypeScript 5.9, pnpm 10.x, against a
Django and DRF backend. This file owns the work that surrounds a diff.
The subjects are the written plan, the success criteria, and the choice between
a question and a stated assumption. They also include the neighbouring file that
already settled a convention, the diff that carries only the request, and the
smallest change that answers it. The last subjects are the orphan that a change
created, and the request that this skill refuses. They also include the decision
that earns a record, and the summary that closes the work.

The installed version, the unconfirmed function, and the four gates are
`references/version-proof-and-unconfirmed-code.md`. The files that instruct an
agent are `references/instruction-files-and-skill-discovery.md`. The commit
message, the hooks, and the lockfile are
`references/dependencies-and-git-workflow.md`. The pipeline stages, the
acceptance criteria, and the report that each command produces are
`references/merge-gates-and-quality-signals.md`.

## Principle

A plan is a cheap experiment. A diff is an expensive one. Write the plan.

A question costs one message. A wrong guess about the architecture costs a
rewrite, and it costs the trust that the next answer needs.

Every changed line is a claim that the request asked for it. A line that
answers no request is a line that a reviewer must read for nothing.

A large diff for a small request hides the real change inside itself. The
reviewer then approves the whole of it, or reads none of it.

An abstraction with one caller is a cost that carries no benefit yet. The
second caller may never arrive, and the cost stays.

Code that nobody asked for still needs a review, a test, and a maintainer. The
person who did not ask for it pays for all three.

A refusal with a reason and an alternative is help. A silent compromise is a
defect with a delay on it.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### The plan comes before the diff

```text
Wrong: the answer opens with a file edit.
Failure: the reader cannot tell which behavior the change targets, so the
review compares the diff against a guess. Where the goal was wrong, the whole
diff is waste, and nobody finds that out until the end.
```

```text
Correct: four parts, under fifteen lines, before any edit.

Goal      One sentence. The behavior that exists after this change.
Success   The commands that must pass, and the states that must render.
Assumed   Each choice that the request left open, and the value taken.
Steps     The ordered edits, each one naming the file it touches.
```

The plan is short because it is a contract and not a document. Four parts hold
it. A plan past fifteen lines is a design document, and the request did not ask
for one.

`README.md` in this repository holds a worked plan for a paginated table. Read
it for the shape that the domain facts take under `Assumed`.

A goal that names a file rather than a behavior is not a goal. `Add a hook to
orders.ts` is a step. `The orders page keeps its page number across a reload`
is a goal.

### Ask where the answer changes the architecture

| The condition | The action | Why the rule holds, and when it flips |
| --- | --- | --- |
| The answer changes the route tree, the data model, or a contract that a user can observe | Ask, and write no code until the answer arrives | A wrong choice here needs a rewrite, and the question needs one message. It flips to the next row where a small diff reverses the choice. |
| A neighbouring file, an existing type, or a convention already answers it | Take that answer, state it under `Assumed`, and proceed | The repository answered the question already, so a question wastes a turn. It flips to the row above where two neighbours disagree. |
| Two readings differ only in a name, an order, or a place | Take the simpler one, and state it | Neither reading is wrong. A question about a reversible choice buys nothing. |
| A fourth question is forming | Stop, and ask the one question that changes the architecture | Four questions in one answer read as an interview. Infer the rest from the repository. |

State an assumption where the request left a choice open. Write it as the value
taken, and never as a question in the past tense.

```text
Wrong: the assumption is a question that nobody answered.
Failure: the reader cannot tell whether the code took a decision or waited for
one, so the review has to reconstruct it from the diff.

  Not sure whether the filter should live in the URL.
```

```text
Correct: the assumption is the value taken, with the reason in one clause.

  Assumed: the filter lives in the URL, because the orders page already
  carries its page number there.
```

### Read the neighbours before you generate

Open the files beside the target before you write. Read the naming, the folder,
the import style, and the shape of the nearest component of the same kind.

Match what you find, even where you would write it differently. A file that
holds two conventions costs more to read than either convention costs to
accept.

`references/directory-and-module-boundaries.md` owns where a file goes.
`references/lint-format-and-scripts.md` owns the rules that a tool enforces.
This file owns the rule that an agent reads the neighbours first, because no
tool enforces a convention that only a person wrote.

Three conventions have no lint rule, and each one needs a read:

- The name of the exported member, against the sibling feature.
- The place where a feature keeps its query options and its schema.
- The wording of a user-facing string, against the nearest one of the same kind.

### Every changed line traces to the request

```tsx
// Wrong: the task was one word in a label, and the tool reformatted the file.
// Failure: the diff shows 180 changed lines, so the reviewer cannot find the
// one that matters. A defect inside the reformat then ships with the label.
export function InviteRow({ invite }: { invite: Invite }) {
  return <label htmlFor="email">Email</label>;
  // ...plus 180 lines that the formatter moved
}
```

```tsx
// Correct: one line changed, and the formatter runs as its own change.
export function InviteRow({ invite }: { invite: Invite }) {
  return <label htmlFor="email">Email</label>;
}
```

Four kinds of change enter a diff that nobody asked for, and each one is a
defect in this domain:

- A whole-file reformat, from an editor that formats on save.
- A rename that improves a name the request never named.
- A comment rewritten for style.
- A move of a file that the request did not ask to move.

Where the repository needs a reformat, run it as its own change with its own
message. `references/lint-format-and-scripts.md` owns the formatter and the
command that runs it over the repository.

### The smallest change that answers the request

```ts
// Wrong: a factory, an interface, and a registry for one caller.
// Failure: three files and a type parameter carry one function call. Every
// later reader has to follow the indirection to learn that one branch exists.
interface Formatter<T> { format(value: T): string }
class FormatterRegistry { /* ...one entry... */ }
export const registry = new FormatterRegistry();
```

```ts
// Correct: the function that the one caller needs.
export function formatOrderTotal(cents: number, locale: string): string {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency: "EUR",
  }).format(cents / 100);
}
```

Three things enter a change under the name of a future need, and none of them
belongs there:

- A configuration option that no caller sets.
- A branch for a state that the types make impossible.
- A layer of indirection that one call site uses.

Where a change reaches two hundred lines and fifty would answer the request,
write the fifty. `references/component-composition.md` owns the size at which a
component splits, and this file owns the rule that the request sets the size.

### Only the orphans that this change created

Remove the import, the variable, and the type that this change left unused.
Leave the code that was already unused, and name it in the summary.

```text
Wrong: the change deletes a helper that nobody in this request mentioned.
Failure: a second feature imported that helper through a barrel, so the build
fails on a file that this request never opened.
```

```text
Correct: the change removes what it orphaned, and reports the rest.

  Removed: the useOrderTotals import, which this change orphaned.
  Not done: formatCents in lib/money.ts has no caller. Left in place.
```

`references/dependencies-and-git-workflow.md` owns the lint rule that reports
an unused import. This file owns the rule that a pre-existing orphan is a
finding and never a deletion.

### A request that implies a server change stops here

A frontend task cannot create a serializer field, an endpoint, a status code, a
permission class, or a migration. Where the request needs one, stop and report
it. Never edit server code, and never state what the server does without the
schema.

Report three things:

1. The exact contract change, named as a field, a route, or a status code.
2. The sibling skill that owns it.
3. The frontend work that the change unblocks.

| The change that the request needs | The sibling skill that owns it |
| --- | --- |
| A serializer field, a route, a status code, a pagination or an error envelope | `django-api-contract` |
| A permission class, a rate limit, or a server-side CSRF rule | `secure-code-auditor` |
| A schema change, a data migration, or a backfill | `django-migration-safety` |
| A Celery task, a queue, or a Channels consumer | `django-async-jobs` |
| A query that costs too much, or a server cache setting | `django-performance-optimizer` |
| A Gunicorn or an ASGI process, or a Django health check | `django-release-readiness` |
| A pytest-django test, a factory, or a fixture | `django-test-auditor` |

`references/openapi-schema-and-codegen.md` owns the generated client and the
command that regenerates it. This file owns the rule that a missing field stops
the work rather than producing a guess.

### Push back where the request is wrong

Three subjects carry an obligation to refuse. They are security,
accessibility, and a change that the team cannot maintain. Name the problem,
give the reason, and offer the correct version.

```text
Wrong: the request wins, and the answer says nothing.
Failure: the interface ships with a keyboard trap, because the request asked
for a custom dropdown and nobody raised the criterion it fails.
```

```text
Correct: the answer names the criterion, and it offers the version that meets
it.

  The div-based dropdown fails WCAG 2.2 criterion 2.1.2, so a keyboard
  reaches it and never leaves. The Base UI select carries the same visual
  design and the full keyboard contract. I built that one.
```

The conflict rule in `SKILL.md` orders the levels: security, then
accessibility, then correctness, then performance, then developer convenience.
Where a requirement forces a trade down that order, state it and stop.

A refusal is not a lecture. One sentence names the failure, one names the
criterion, and one offers the alternative. `accessibility-wcag` and
`frontend-security` hold a veto, so a task that fails either one is a failed
task.

### The decision record, and when to write none

```text
Correct: five parts, in one file under docs/decisions/.

  Title          0007 — the orders list holds its filters in the URL
  Status         Accepted
  Context        Two readers share a filtered list by pasting a link.
  Decision       nuqs holds the filter, the sort, and the page.
  Consequences   A filter change writes history. The cache key derives
                 from the URL, so no second store holds the same value.
```

Write a record where the choice is expensive to reverse. Four choices qualify:

- Where a value lives, between the URL, the server, and a client store.
- The consistency model between the cache and a pushed event.
- One application against a workspace of packages.
- A library that the interface cannot easily leave.

Write no record for a name, a folder, a class order, or any choice that a small
diff reverses. A collection of trivial records buries the load-bearing ones,
and a reader then trusts none of them.

The status moves from Proposed to Accepted, and later to Deprecated or
Superseded. A record is never edited after it is accepted. A new record
supersedes it, and it names the number that it replaces.

`references/directory-and-module-boundaries.md` owns the workspace decision
itself, and it states that the decision belongs in `AGENTS.md`.
`references/instruction-files-and-skill-discovery.md` owns that file. This file
owns the record and the rule that selects it.

### The summary that closes the work

```text
Wrong: the answer ends at the last edit.
Failure: the reviewer cannot tell what the change assumed, so every assumption
becomes a question in the review. Nothing states what the change left undone,
so the gap reaches production as a surprise.
```

```text
Correct: three lines close the work.

  Changed    The orders page reads the filter from the URL. 4 files.
  Assumed    The filter is a single status value, because the DRF schema
             carries one status field and no list form.
  Not done   The saved-view feature. It needs a new endpoint, which
             django-api-contract owns.
```

`Not done` is the part that a reader cannot reconstruct. It holds the request
that this change did not answer, the pre-existing orphan that stayed, and the
domain check that the change did not reach.

`references/merge-gates-and-quality-signals.md` owns the report of each command
and its output. This file owns the three lines above it.

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| The review compares the diff against a guessed goal | The answer carried no plan | Read the answer for a goal and a success line | Write the four-part plan, and restate the goal |
| The architecture is wrong, and the code is correct | An architectural question was answered in silence | The plan states one reading where two existed | Name both readings, and ask the one question |
| The reader answers four questions the repository already held | No neighbouring file was read | Count the questions in the answer | Read the neighbours, and ask only the architectural one |
| A defect ships inside a reformat | The editor formatted on save | `git diff --stat` shows churn far past the request | Revert the reformat, and run it as its own change |
| One call site carries three files of indirection | An abstraction was written for a second caller that never arrived | Count the callers of the new type | Inline it, and keep the one function |
| The build fails in a file that the request never named | A pre-existing orphan was deleted | `git diff` lists a file outside the request | Restore it, and report it under `Not done` |
| The client compiles against a field that the API does not carry | An implied server change was guessed rather than reported | Compare the type against the generated client | Revert, name the contract change, and route it |
| The interface ships with a keyboard trap | A wrong request produced silence | Read the answer for the criterion that the request fails | State the criterion, and build the version that meets it |
| A settled decision returns in the next session | The choice carries no record | Read `docs/decisions/` for the subject | Write the record, and link it from the code |
| Every assumption becomes a question in the review | The work closed with no summary | Read the last lines of the answer | Add the `Changed`, `Assumed`, and `Not done` lines |

## Verification

```bash
# 1. Read the plan against the diff. Every step must name a file that changed.
git diff --name-only

# 2. Read the churn per file. A count far past the request is a drive-by edit.
git diff --stat

# 3. Read the whole diff for a reformat, a rename, or a moved comment.
git diff

# 4. Find a file that the request never named. Read every hit.
git diff --name-only | rg -v 'orders|invite'

# 5. Confirm that no server file entered the diff. This prints nothing.
git diff --name-only | rg '\.py$|/migrations/|settings/'

# 6. Count the callers of a type that this change introduced.
rg -n --count-matches 'FormatterRegistry' src/

# 7. Find an unused export that this change created.
rg -n 'export (const|function) formatOrderTotal' src/ \
  && rg -n 'formatOrderTotal' src/ --files-with-matches

# 8. Read the decision records, and confirm that none is edited after Accepted.
rg -n '^Status' docs/decisions/

# 9. Confirm that the answer closes with the three lines.
#    Read the answer. Changed, Assumed, and Not done must each be present.
```

## Review checklist

- [ ] Does a written plan with a goal, a success line, the assumptions, and the
      steps come before the first edit?
- [ ] Is the plan under fifteen lines?
- [ ] Does the goal name a behavior rather than a file?
- [ ] Was every architectural question asked, rather than answered in silence?
- [ ] Does each open choice appear under `Assumed` as the value taken?
- [ ] Are there three or fewer clarifying questions in the answer?
- [ ] Were the neighbouring files read, and does the new code match their
      naming, their folder, and their import style?
- [ ] Does every changed line trace to the request?
- [ ] Is the diff free of a whole-file reformat, an unrequested rename, a
      rewritten comment, and a moved file?
- [ ] Is the change the smallest one that answers the request?
- [ ] Does every new type, option, and layer have at least one caller?
- [ ] Were only the orphans that this change created removed?
- [ ] Is every pre-existing orphan reported rather than deleted?
- [ ] Does the diff touch no server file?
- [ ] Is every implied contract change named, and routed to the sibling skill
      that owns it?
- [ ] Was a request that fails security, accessibility, or maintainability
      refused with a reason and an alternative?
- [ ] Does every choice that a small diff cannot reverse carry a decision
      record?
- [ ] Is every trivial choice free of a decision record?
- [ ] Does the answer close with the `Changed`, `Assumed`, and `Not done`
      lines?

## Handoffs

- The installed version, the unconfirmed function, the four gates, and the
  calibration of a claim →
  `references/version-proof-and-unconfirmed-code.md`.
- `AGENTS.md`, `CLAUDE.md`, the managed Next.js block, and the audit of a
  third-party skill → `references/instruction-files-and-skill-discovery.md`.
- Where a file goes, the barrel rule, and the workspace decision →
  `references/directory-and-module-boundaries.md`.
- The commit message, the git hooks, the lockfile, and the unused-import rule →
  `references/dependencies-and-git-workflow.md`.
- The formatter, the lint command, and the script that runs them →
  `references/lint-format-and-scripts.md`.
- The pipeline stages, the acceptance criteria, and the output of each command →
  `references/merge-gates-and-quality-signals.md`.
- The test that reproduces a report before the fix →
  `references/test-strategy-and-component-tests.md`.
- The generated client, and the command that regenerates it →
  `references/openapi-schema-and-codegen.md`.
- The criterion that a control fails, and the keyboard contract →
  `references/wcag-conformance-and-verification.md` and
  `references/keyboard-focus-and-live-regions.md`. That domain holds a veto.
- The sink, the sanitiser, and the secret that must not reach the browser →
  `references/untrusted-markup-and-injection.md` and
  `references/secret-boundary-and-supply-chain.md`. That domain holds a veto.
- The size at which a component splits →
  `references/component-composition.md`.
- The serializer, the route, the status code, and the envelope on the server →
  the sibling skill `django-api-contract`. This file owns the report only.
- The permission class, the rate limit, and the server-side CSRF rule → the
  sibling skill `secure-code-auditor`.
- The migration, the backfill, and the table lock → the sibling skill
  `django-migration-safety`.
- The Celery task, the queue, and the Channels consumer → the sibling skill
  `django-async-jobs`.
- The query cost and the server cache setting → the sibling skill
  `django-performance-optimizer`.
- The Django process, the server config, and the go-live gate → the sibling
  skill `django-release-readiness`.
- The pytest-django test, the factory, and the fixture → the sibling skill
  `django-test-auditor`.
