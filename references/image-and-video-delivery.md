# Image and video delivery

Next.js 16.3 with `next/image`, React 19.2.6 or later, `hls.js`,
`shaka-player`, `lite-youtube-embed`, against a Django and DRF backend. This
file owns the bytes of a picture and of a moving picture, from the source to the
screen. The subjects are the reserved box, the source set, and the image that
paints the largest area. They also include the optimizer that produces the
variants, and the object URL behind a preview. The last subjects are the
`<video>` and `<audio>` elements with their captions, and the facade in front
of a third-party player.

The path that puts the file into storage is
`references/file-upload-and-transport.md`. The origin that serves a stored file
back is `references/served-content-and-downloads.md`.

## Principle

A picture with no stated box has a height of zero until its bytes arrive. Every
element below it then moves, and the user is reading while it moves.

A browser picks a variant before it knows the layout. It assumes the full width
of the viewport unless the markup states otherwise, and it downloads more bytes
than the screen can show.

The element that paints the largest area decides the time that a user waits. The
browser cannot request it early unless the first document names it.

A blob URL holds its blob in memory until the code releases it. A list that
creates one for each row and releases none holds every file that the user
opened.

A moving picture without captions excludes anyone who cannot hear it, and anyone
in a room where sound is not possible.

A third-party player costs hundreds of kilobytes before a user asks for it. A
picture and a button cost almost nothing, and they buy the same result.

## Pinned-stack depth

Each recommendation in this file is current practice at the versions above,
unless the text gives it a different mark. The two other marks are current but
in decline, and alive only in legacy code.

### Every image states its box, and `sizes` states its width

```tsx
// Wrong: no dimensions, and a lazy hero.
// Failure: the layout reserves no height, so the page moves when the bytes
// arrive. The hero also carries loading="lazy", so the browser finds it after
// the layout rather than in the first document, and the largest paint is late.
<img src="/hero.jpg" loading="lazy" />
```

```tsx
// Correct: the box is reserved, the largest element is preloaded, and sizes
// states the real width.
import Image from "next/image";

<Image src="/hero.jpg" alt="" width={1920} height={1080} preload sizes="100vw" />;

<Image
  src={product.image}
  alt={product.name}
  fill
  sizes="(max-width: 768px) 100vw, 33vw"
  className="object-cover"
/>;
```

Give every image an explicit `width` and `height`, or `fill` with a parent that
holds `position: relative` and a reserved ratio. A user upload and a CMS record
carry no intrinsic size at build time, so they take `fill`.
`references/layout-and-typography.md` owns the CSS that reserves the ratio.

Set `sizes` on every responsive image and on every `fill` image. Derive the
value from the layout, and not from a guess. A missing `sizes` makes the browser
assume `100vw`, so a phone downloads the variant for a desktop.

A raw `<img>` needs a written reason. The reason is a real one for an SVG icon
in the bundle, or for a source that the optimizer must not touch. It is never
"the config raised an error".

### The largest element is preloaded, and never lazy

Next 16 deprecated the `priority` prop of `<Image>` in favour of `preload`. The
two express the same intent, and `preload` is the current name.

Set `preload` on the one element that paints the largest area, which is usually
the hero image. Where the layout has more than one candidate across the
breakpoints, set `fetchPriority="high"` instead, so no request is wasted on a
picture that stays hidden.

NEVER put `loading="lazy"` on that element. A lazy attribute moves discovery out
of the first document, and the largest paint waits for the layout.

The `preload` prop of `<Image>` is not the `preload` function of `react-dom`.
`references/suspense-and-actions.md` owns that function.

### The optimizer needs a scope, and SVG stays out of it

```ts
// Wrong: the optimizer accepts every host, and it renders SVG.
// Failure: any URL becomes a request from the server, which is a request
// forgery surface. A crafted SVG then costs the process its CPU and its memory,
// which CVE-2026-64644 describes.
images: {
  remotePatterns: [{ hostname: "**" }],
  dangerouslyAllowSVG: true,
}
```

```ts
// Correct: a named host, a named path, and no SVG through the optimizer.
images: {
  remotePatterns: [
    { protocol: "https", hostname: "media.example.com", pathname: "/uploads/**" },
  ],
}
```

Next.js skips the optimizer by itself when a `src` ends in `.svg`, and the
`unoptimized` prop states the same intent. Serve a user-supplied SVG from
another origin, which `references/served-content-and-downloads.md` states, or
convert it to a raster image on the server.

The optimizer is expensive in CPU and in memory, and its disk cache grows until
the limit removes the oldest entries. A self-hosted deployment needs `sharp`
present. On a small server, hand the work to a CDN or to a `loaderFile` instead,
and confirm the cache time.

`references/app-router-structure.md` owns `next.config.ts` and the Next 16
default changes, which include `minimumCacheTTL` at 14400 seconds, `qualities`
at `[75]`, and `maximumRedirects` at 3. Read that file before you set a
`quality` prop, because a value outside `qualities` moves to the nearest allowed
one.

### An object URL is released

```tsx
// Wrong: the render creates a URL for each item, on each pass.
// Failure: every blob stays in memory for the lifetime of the document. A
// gallery that the user opens and closes ten times holds ten copies.
{files.map((f) => <img key={f.name} src={URL.createObjectURL(f)} />)}
```

```tsx
// Correct: one URL for one file, released when the component unmounts.
import { useEffect, useState } from "react";

function Preview({ file }: { file: File }) {
  const [url, setUrl] = useState<string>();

  useEffect(() => {
    const objectUrl = URL.createObjectURL(file);
    setUrl(objectUrl);
    return () => URL.revokeObjectURL(objectUrl);
  }, [file]);

  return url ? <img src={url} alt="" className="size-24 object-cover" /> : null;
}
```

Every `URL.createObjectURL` call has a matching `URL.revokeObjectURL` call. The
cleanup of the effect is the place for it, and
`references/state-and-effects.md` owns the rule that an effect returns a
cleanup.

### The video, the audio, and the captions

Take a plain `<video>` with a single MP4 source where one file serves every
viewer. Give it `controls`, a `poster` so the box is not empty, and
`playsInline` so a phone plays it in place.

Adaptive streaming needs a player. Safari plays HLS by itself. Chrome, Firefox,
and Edge need a JavaScript player over Media Source Extensions. Take `hls.js`
for HLS, and take `shaka-player` for DASH, for HLS, and for protected content.
Full CEA-708 captions need `shaka-player`. Load the player on demand, because
its bundle is large.

Every `<video>` carries a `<track kind="captions">` with a WebVTT file, or the
page carries a transcript beside it. State which one the product ships before
the player is written. A custom control set is a set of real buttons that a
keyboard reaches, and `references/keyboard-focus-and-live-regions.md` owns that
contract.

Autoplay needs `muted`. A browser refuses to start sound that a user did not
ask for.

### A facade replaces a third-party player

```tsx
// Correct: a picture and a button first, and the real player after a click.
"use client"; // it swaps the picture for an iframe on a click

import { useState } from "react";

type FacadeProps = { embedSrc: string; posterSrc: string; title: string };

function PlayerFacade({ embedSrc, posterSrc, title }: FacadeProps) {
  const [play, setPlay] = useState(false);

  if (play) {
    return (
      <iframe
        title={title}
        allow="autoplay; encrypted-media"
        allowFullScreen
        src={embedSrc}
        className="aspect-video w-full"
      />
    );
  }

  return (
    <button type="button" onClick={() => setPlay(true)} className="aspect-video w-full">
      <span className="sr-only">{`Play ${title}`}</span>
      <img src={posterSrc} alt="" loading="lazy" />
    </button>
  );
}
```

A standard embed loads a player, its analytics, and its cookies before anybody
presses play. The facade loads a picture and a button. The cost is that autoplay
on arrival is no longer possible.

Build `embedSrc` against the `youtube-nocookie.com` host rather than the
`youtube.com` host, so the frame sets no tracking cookie. Give the `<iframe>` a
`title`, because the frame is a region that a screen reader lists.

`lite-youtube-embed` ships this pattern as a component. The frame source of a
facade and the origins that it needs belong in the Content Security Policy,
which is `references/security-headers-and-csp.md`.

### The libraries

The table gives each package its latest version, its last release date, its
maintenance status, and its open advisories. The package registry and the
advisory database supplied those facts on 16 August 2026. A cell that holds no
date is a package with a current registry entry on that date. This file does not
state an exact release date for it.

| Tier | Package | The rule | Latest version | Last release | Maintenance | Open advisories |
| --- | --- | --- | --- | --- | --- | --- |
| Recommend | `next/image` | The default path for every image. Keep Next.js patched, because the optimizer carries its own advisories. | Next.js 16.3 | Current | Active, by Vercel | None on 16.3 |
| Recommend | `hls.js` | HLS playback where the browser does not carry it. | Current | Current | Active | None |
| Recommend | `shaka-player` | DASH, HLS, protected content, and full CEA-708 captions. | 5.x | Current | Active, by Google | None |
| Recommend | `lite-youtube-embed` | The facade in front of a YouTube player. | Current | Current | Active | None |
| Conditional | `mux-player` or `video.js` | Only where the product needs a managed player interface. Both weigh more than `<video>` with `hls.js`. | Current | Current | Active | None |
| Conditional | A CDN loader through `loaderFile` | Only where the deployment cannot pay the CPU and the disk of the optimizer. | — | — | — | — |
| Audit-only | `next/legacy/image` | Deprecated. Move to `next/image`. | — | — | — | — |
| Reject | `images.domains` | Replaced by `images.remotePatterns`. | — | — | — | — |
| Reject | `dangerouslyAllowSVG` over user content | It opens a script and a denial-of-service surface, which CVE-2026-64644 describes. | — | — | — | — |

### Version discipline

Read the installed versions before you write code.

React 19.2.6 is the security floor, for the reason that
`references/state-and-effects.md` states.

| The change | Detect the stale idiom | The migration |
| --- | --- | --- |
| Next 16 deprecated the `priority` prop of `<Image>` | `rg 'priority' -g '*.tsx'` reports a hit on an `<Image>` | Take `preload`. `@next/codemod` carries the change |
| Next 16 set `images.imageSizes` without `16`, so the smallest variant is gone | A layout that renders an image at 16 CSS pixels | Add `16` back to `imageSizes`, or render the icon as an SVG in the bundle |
| CVE-2025-59471, an out-of-memory defect in the optimizer, fixed in 15.5.10 and 16.1.5 | `npm ls next` reports a version below the patch on its line | Upgrade, and narrow `remotePatterns` |
| CVE-2026-64644, a denial of service through an SVG in the optimizer, fixed in 15.5.21 and 16.2.11 | `npm ls next` reports a version below the patch on its line | Upgrade. Until it lands, set `experimental.imgOptSkipMetadata: true` |

## Verification

```bash
# 1. Read the installed versions before you write code.
rg -n '"next"|"hls.js"|"shaka-player"|"sharp"' package.json

# 2. Find a raw img tag. Each hit needs a written reason.
rg -n '<img' -g '*.tsx' src/

# 3. Find a fill image. Read every hit, and confirm a sizes prop within a few
#    lines.
rg -n -A6 '\bfill\b' -g '*.tsx' src/

# 4. Find a lazy attribute. No hit is the element that paints the largest area.
rg -n 'loading="lazy"' -g '*.tsx' src/

# 5. Find the deprecated prop on an Image. This prints nothing.
rg -n -B4 '\bpriority\b' -g '*.tsx' src/

# 6. Find an unscoped host and an SVG exception. This prints nothing.
rg -n "hostname:\s*['\"]\*\*|dangerouslyAllowSVG" next.config.ts

# 7. Find an unoptimized prop. Each hit needs a written reason.
rg -n 'unoptimized' -g '*.tsx' src/

# 8. Find a file that creates an object URL and never releases one.
rg --files-with-matches 'createObjectURL' -g '*.ts*' src/ \
  | xargs rg --files-without-match 'revokeObjectURL'

# 9. Find a video element with no captions track and no transcript.
rg --files-with-matches '<video' -g '*.tsx' src/ \
  | xargs rg --files-without-match 'kind="captions"'

# 10. Find a third-party player frame with no facade around it.
rg -n 'youtube\.com/embed|player\.vimeo\.com' -g '*.tsx' src/

# 11. Run Lighthouse on a page that holds pictures. It reports no oversized
#     image, no layout shift from media, and the largest element in the first
#     document.

# 12. Open a gallery ten times, then read the memory panel of the browser.
#     The retained blob count does not climb.
```

## Review checklist

- [ ] Does every image use `next/image`, or carry a written reason not to?
- [ ] Does every image state `width` and `height`, or `fill` with a reserved
      ratio?
- [ ] Does every responsive image and every `fill` image state `sizes`, derived
      from the layout?
- [ ] Does the element that paints the largest area carry `preload`, or
      `fetchPriority="high"` where the layout has several candidates?
- [ ] Is `loading="lazy"` absent from that element?
- [ ] Is `priority` absent from every `<Image>`, in favour of `preload`?
- [ ] Is `remotePatterns` scoped to named hosts, with no `**` hostname?
- [ ] Is `dangerouslyAllowSVG` off?
- [ ] Does a self-hosted deployment carry `sharp`, or hand the work to a CDN?
- [ ] Does every `URL.createObjectURL` call have a matching
      `URL.revokeObjectURL` call in a cleanup?
- [ ] Does every `<video>` carry a captions track, or does the page carry a
      transcript?
- [ ] Does a keyboard reach every player control?
- [ ] Does a third-party player sit behind a facade?
- [ ] Is the player bundle loaded on demand, rather than in the first payload?

## Handoffs

- The CSS that reserves the ratio, the skeleton geometry, and the fluid type
  scale → `references/layout-and-typography.md`.
- The `next.config.ts` keys, the Next 16 default changes, and the route files →
  `references/app-router-structure.md`.
- The `"use client"` directive over a player island, and the boundary around it →
  `references/server-and-client-components.md`.
- The cleanup that an effect returns → `references/state-and-effects.md`.
- The `preload` function of `react-dom`, and the fallback around a slow media
  panel → `references/suspense-and-actions.md`.
- The alternative text of an image, the `role="img"` on an inline SVG, and the
  name of a play control → `references/semantics-and-accessible-names.md`. That
  domain holds a veto.
- The keyboard contract of a custom player, and the announcement of a state
  change → `references/keyboard-focus-and-live-regions.md`. That domain holds a
  veto.
- The contrast of a control over a picture, the target size, and the
  reduced-motion preference → `references/visual-and-motor-criteria.md`. That
  domain holds a veto.
- The tokens behind a player control, and the dark theme over a poster →
  `references/design-tokens-and-theming.md`.
- The picked file that a preview renders →
  `references/file-upload-and-transport.md`.
- The origin, the `Content-Disposition` header, and the `nosniff` header over a
  stored image → `references/served-content-and-downloads.md`.
- The Content Security Policy that names a frame source and a media source →
  `references/security-headers-and-csp.md`. The reason that a wildcard host in
  `remotePatterns` is a server-side request forgery is
  `references/exposed-endpoints-and-destinations.md`. That domain holds a veto.
- The largest paint and the layout shift →
  `references/paint-and-interaction-cost.md`. The budget over the bundle of a
  player → `references/client-bundle-and-third-party-scripts.md`.
- The transition between two pictures, and the reduced variant behind it →
  `references/motion-primitives-and-reduced-motion.md`.
- The words of a caption and of a transcript →
  `references/interface-copy-and-voice.md`. The copy of an empty media state →
  `references/error-and-empty-state-copy.md`.
- The Open Graph image and the metadata around it →
  `references/route-metadata-and-social-cards.md`.
- The memory and the disk that the optimizer costs on a host →
  `references/runtime-process-and-reverse-proxy.md`. The `sharp` module inside a
  container image → `references/build-output-and-container-image.md`.
- The transcode job and the thumbnail that it produces → the backend.
