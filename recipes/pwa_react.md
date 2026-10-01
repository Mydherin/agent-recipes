---
name: Native-feeling PWA for Safari iOS and Chrome Android
description: Turn a responsive React + Vite + Tailwind SPA into an installable PWA that behaves like a native app on iOS Safari and Android Chrome — keyboard-safe sheets, non-overlapping layers and dropdowns, safe areas, no browser chrome or callouts, correct icons, offline-safe launch and instant updates.
---

# Native-feeling PWA for Safari iOS and Chrome Android

## Goal

Ship one web codebase that, once added to the home screen, is indistinguishable from a native app on iPhone/iPad (Safari/WebKit) and Android (Chrome):

- Launches full screen from its own icon, offline-safe, and always on the latest deploy.
- The on-screen keyboard never hides the field being typed into, nor the actions of a sheet.
- Every layer (app bar, menus, drawers, modals, FAB, toasts) stacks predictably; no dropdown is clipped or overlaps another.
- No browser artefacts: no zoom on focus, no double-tap zoom, no long-press callouts, no text selection on controls, no tap flashes, no rubber-band leaking behind modals, no stuck hover states.
- Content respects notches, the home indicator and rounded corners in portrait and landscape.

## Scope and assumptions

- Stack: React SPA with client-side routing, built by Vite (content-hashed assets under `/assets/`), styled with Tailwind CSS (v4) plus plain CSS where the rule is global.
- Targets: current iOS/iPadOS Safari (17+) installed via *Add to Home Screen*, current Chrome on Android installed via the install prompt; desktop Chrome/Edge/Safari must keep working.
- Served over HTTPS (localhost is exempt) by any static server with an SPA fallback; the API lives under a same-origin prefix (`/api/` below).
- Snippets use generic names (`.app-bar`, `.sheet`, `.menu`, `.drawer`, `.fab`, `.toast-stack`). Map them to the project's components; the CSS rules are what matter.

## Requirements

### Functional

- **Installable** on Android Chrome (install prompt / menu) and iOS Safari (Share → Add to Home Screen), opening in `standalone` mode with the app's name, icon and colours.
- **Icons**: crisp favicon in browser tabs, correct home-screen icon on iOS (opaque 180×180), adaptive/maskable icon on Android, no white or black frames.
- **Offline launch**: opening the installed app without network shows the app shell (last known version) instead of a browser error page; API calls fail gracefully inside the UI.
- **Updates**: a new deploy is picked up on the next launch without the user reinstalling or clearing data.
- **Keyboard**: focusing a field inside a sheet/modal lifts the sheet above the keyboard and scrolls the field into view; toasts stay visible above the keyboard; opening a modal on touch does not pop the keyboard by itself.
- **Layers**: modals, drawers, menus, FAB and toasts follow a single z-order scale; overlays render above every page stacking context.
- **Dropdowns**: selects use the native OS picker on touch; custom result lists never overlap other controls or each other; at most one popover open at a time; outside tap and `Escape` close it.
- **Resume**: returning to the app (from background or the home screen) refreshes time-sensitive data, since standalone apps have no pull-to-refresh.
- **Shortcuts** (Android): long-press on the icon offers deep links to the main sections.

### Non-functional

- No `user-scalable=no` / `maximum-scale`: pinch zoom stays available for accessibility; focus zoom is prevented by font size instead.
- Touch targets ≥ 44×44 px; layouts work from 320 px width up, portrait and landscape.
- Service worker and manifest are never HTTP-cached; hashed assets are cached immutably; HTML shell is revalidated on every launch.
- The service worker never touches API traffic, non-GET requests or third-party origins, and is disabled in development.
- No per-platform user-agent sniffing; behaviour is driven by feature queries (`pointer`, `hover`, `@supports`, `visualViewport`).
- Viewport tracking is rAF-throttled and costs nothing while idle.

## Approach

Treat the installed PWA as a native shell built from five independent layers. Each layer is self-contained so it can be adopted, reviewed and regressed separately:

1. **Identity** — HTML head + web manifest + icon set: how the OS presents the app.
2. **Delivery** — service worker + server cache headers: how the app launches offline and updates.
3. **Viewport** — safe areas, dynamic viewport units and a tiny script that mirrors the *visual* viewport and keyboard height into CSS variables. This is the core trick: iOS never resizes the layout viewport for the keyboard and Chrome only shrinks the visual one, so `position: fixed` UI must be positioned against measured values, not against `100vh`/`bottom: 0`.
4. **Interaction model** — global CSS that removes browser behaviours a native app does not have (zoom on focus, callouts, selection, tap highlight, sticky hover, scroll chaining) and adds touch press feedback.
5. **Overlay system** — one z-index scale, the native `<dialog>` top layer for modals, portals for drawers, bottom-sheet layout on phones, and dropdown rules that make overlap impossible by construction.

Key decisions:

- **Measure, do not guess, the keyboard**: `visualViewport` works identically on WebKit and Blink. Do not rely on `interactive-widget=resizes-content` (Chrome-only) or the VirtualKeyboard API (Chrome-only); they would make Android and iOS behave differently.
- **Prefer platform primitives**: `<dialog>.showModal()` (top layer, focus trap, `Escape`/Android back gesture → `cancel`), native `<select>` (OS picker never clipped), `env(safe-area-inset-*)`, `dvh`.
- **Hand-written service worker** (~50 lines) instead of a plugin: the strategy is simple, fully auditable, and Vite already fingerprints assets. A plugin such as `vite-plugin-pwa`/Workbox is an acceptable substitute if it reproduces exactly the strategy in step 2.

## Steps

### 1. Identity: HTML head

Place in `index.html` (order matters only for readability):

```html
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<meta name="description" content="<App> — <tagline>" />
<meta name="color-scheme" content="light" />              <!-- or "light dark" if the app themes both -->
<meta name="theme-color" content="#ffffff" />             <!-- = app bar background -->
<meta name="application-name" content="<App>" />
<meta name="mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-capable" content="yes" /> <!-- legacy iOS, harmless -->
<meta name="apple-mobile-web-app-title" content="<App>" /> <!-- home-screen label, ≤ 12 chars -->
<meta name="apple-mobile-web-app-status-bar-style" content="default" />
<meta name="format-detection" content="telephone=no" />
<link rel="manifest" href="/manifest.webmanifest" />
<link rel="icon" href="/favicon.svg" type="image/svg+xml" />
<link rel="icon" href="/favicon-32.png" type="image/png" sizes="32x32" />  <!-- fallback for engines without SVG favicons -->
<link rel="apple-touch-icon" href="/icons/apple-touch-icon.png" />
```

Rules:

- `viewport-fit=cover` is mandatory: without it `env(safe-area-inset-*)` is always `0` and landscape iPhones show letterbox bars.
- Never add `maximum-scale=1` or `user-scalable=no` (accessibility failure; iOS ignores it anyway). Focus zoom is solved in step 4.
- `theme-color` must equal the app bar background and `background_color` must equal the page background so status bar, splash and first paint blend without flashes.
- `status-bar-style`: `default` keeps content below the status bar (simplest). `black-translucent` draws the page under it; only use it when the app bar already pads with `env(safe-area-inset-top)` (step 3 does) and the top colour is dark enough for white status text.
- `format-detection telephone=no` stops iOS turning numbers (IDs, amounts) into tappable phone links.
- `color-scheme` controls native form controls, scrollbars and pickers. Declare exactly the schemes the design supports, otherwise dark-mode devices render dark native controls inside a light UI.
- Web fonts: `preconnect` to the font origin and use `font-display: swap`; always end the `font-family` with a system stack so offline launches still render.

### 2. Identity: web manifest and icons

`public/manifest.webmanifest` (Vite copies `public/` verbatim, so the URL is stable):

```json
{
  "id": "/",
  "name": "<App>",
  "short_name": "<App>",
  "description": "<App> — <tagline>",
  "lang": "en",
  "dir": "ltr",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "orientation": "any",
  "background_color": "#f7f6fb",
  "theme_color": "#ffffff",
  "categories": ["productivity"],
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "/icons/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "shortcuts": [
    { "name": "<Section A>", "short_name": "<A>", "url": "/section-a" },
    { "name": "<Section B>", "short_name": "<B>", "url": "/section-b" }
  ]
}
```

Rules:

- `id` fixes the app identity; never change it after release or installs duplicate.
- `start_url` and `scope` at `/` so every client route stays inside the app window (links outside scope open a browser sheet).
- `orientation: "any"` unless the product truly forbids rotation; the layout must then handle landscape phones (step 3).
- Keep `purpose: "any"` and `purpose: "maskable"` as **separate files**. Never `"any maskable"`: the maskable art looks shrunken where it is used as a plain icon.
- Optional for a richer Android install sheet: `screenshots` (with `form_factor: "narrow"`/`"wide"`).

Icon set (`public/icons/` + root favicon):

| File | Size | Notes |
| --- | --- | --- |
| `favicon.svg` | vector | Browser tab. Own background shape so it reads on light and dark tabs. |
| `favicon-32.png` | 32×32 | Fallback favicon. |
| `apple-touch-icon.png` | 180×180 | **Opaque** (iOS fills transparency with black), square, no rounded corners (iOS masks it). |
| `icon-192.png`, `icon-512.png` | 192, 512 | `any` purpose; may be transparent. |
| `icon-maskable-512.png` | 512×512 | Full-bleed background; logo inside the central 80 % safe circle. Validate with a maskable preview tool. |

For pixel-art or line-art logos export with nearest-neighbour scaling from the vector source to keep edges crisp.

iOS splash: iOS does not reliably build a launch screen from the manifest. Either ship `apple-touch-startup-image` links per device size, or (cheaper, used here) keep the shell tiny so first paint is near instant over a `background` that matches `background_color`.

### 3. Delivery: service worker

`public/sw.js` — served from the root so its scope is `/`:

```js
// App-shell service worker: the installed app launches instantly and offline-safe,
// while API traffic and third-party requests always go straight to the network.
const SHELL_CACHE = 'app-shell-v1'
const ASSET_CACHE = 'app-assets-v1'
const SHELL_URL = '/index.html'

self.addEventListener('install', (event) => {
  event.waitUntil(caches.open(SHELL_CACHE).then((cache) => cache.add(SHELL_URL)).then(() => self.skipWaiting()))
})

self.addEventListener('activate', (event) => {
  event.waitUntil(caches.keys()
    .then((keys) => Promise.all(keys.filter((key) => key !== SHELL_CACHE && key !== ASSET_CACHE).map((key) => caches.delete(key))))
    .then(() => self.clients.claim()))
})

self.addEventListener('fetch', (event) => {
  const { request } = event
  if (request.method !== 'GET') return
  const url = new URL(request.url)
  if (url.origin !== self.location.origin || url.pathname.startsWith('/api/')) return
  if (request.mode === 'navigate') event.respondWith(networkFirstShell(request))
  else if (url.pathname.startsWith('/assets/') || url.pathname.startsWith('/icons/')) event.respondWith(cacheFirst(request))
})

// Every route is the SPA shell, so the latest index.html doubles as the offline fallback.
async function networkFirstShell(request) {
  try {
    const response = await fetch(request)
    if (response.ok && response.type === 'basic') {
      const cache = await caches.open(SHELL_CACHE)
      await cache.put(SHELL_URL, response.clone())
    }
    return response
  } catch (error) {
    const cached = await caches.match(SHELL_URL)
    if (cached) return cached
    throw error
  }
}

// Vite fingerprints asset names, so a cached file never goes stale.
async function cacheFirst(request) {
  const cached = await caches.match(request)
  if (cached) return cached
  const response = await fetch(request)
  if (response.ok && response.type === 'basic') {
    const cache = await caches.open(ASSET_CACHE)
    await cache.put(request, response.clone())
  }
  return response
}
```

Strategy summary:

| Request | Strategy | Why |
| --- | --- | --- |
| Navigation (any route) | Network-first, fallback to cached `index.html` | Always latest deploy online; shell offline. |
| `/assets/*` (hashed), `/icons/*` | Cache-first | Immutable by name. |
| `/api/*`, non-GET, cross-origin | Not intercepted | No stale data, no auth/caching surprises. |

Rules:

- Bump the cache names (`-v2`) only when the SW strategy itself changes; content changes need nothing because of hashing + network-first shell.
- `skipWaiting()` + `clients.claim()` make a new SW take over immediately; safe because the SW holds no app logic.
- Only cache `ok` + `basic` (same-origin, non-opaque) responses.

Registration — `src/shared/pwa/register-service-worker.ts`, called once from the entry file before rendering:

```ts
// Production only: a service worker would serve stale modules to Vite's dev server.
export function registerServiceWorker() {
  if (!import.meta.env.PROD || !('serviceWorker' in navigator)) return
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js').catch(() => undefined)
  })
}
```

Registering after `load` keeps the SW install off the critical path of the first visit.

### 4. Delivery: server headers

Whatever serves `dist/` must apply, in priority order:

| Path | Headers / behaviour |
| --- | --- |
| `/sw.js` | `Cache-Control: no-cache` (browsers also cap SW cache at 24 h, do not rely on it). |
| `/manifest.webmanifest` | `Content-Type: application/manifest+json`, `Cache-Control: no-cache`. |
| `*.js`, `*.css`, fonts (hashed) | `Cache-Control: public, max-age=31536000, immutable`. |
| images | `Cache-Control: public, max-age=2592000`. |
| everything else | SPA fallback to `/index.html` with `Cache-Control: no-cache`. |

nginx reference:

```nginx
location = /sw.js {
    add_header Cache-Control "no-cache" always;
    try_files $uri =404;
}

location = /manifest.webmanifest {
    types { application/manifest+json webmanifest; }
    add_header Cache-Control "no-cache" always;
    try_files $uri =404;
}

location ~* \.(js|css|woff2?|ttf|eot|otf)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    try_files $uri =404;
}

location ~* \.(png|jpg|jpeg|gif|ico|svg|webp)$ {
    expires 30d;
    add_header Cache-Control "public";
    try_files $uri =404;
}

# The shell is revalidated on every launch so installed apps pick up new deploys.
location / {
    add_header Cache-Control "no-cache" always;
    try_files $uri $uri/ /index.html;
}
```

nginx pitfalls: exact matches (`location =`) must exist or the regex `\.js$` rule would cache `sw.js` for a year; any `add_header` inside a `location` drops **all** server-level `add_header` directives (CSP, HSTS…), so repeat them there.

### 5. Viewport: global CSS foundation

In the global stylesheet (after `@import "tailwindcss";`):

```css
:root {
  color-scheme: light;
  /* Live visual viewport, kept current by the viewport tracker (step 6). */
  --viewport-height: 100dvh;
  --keyboard-inset: 0px;
  --app-bar-height: 74px;
  scroll-padding-top: calc(var(--app-bar-height) + env(safe-area-inset-top) + 12px);
  overscroll-behavior-y: none;
  -webkit-text-size-adjust: 100%;
  text-size-adjust: 100%;
  -webkit-tap-highlight-color: transparent;
}

* { box-sizing: border-box; }
body { margin: 0; min-width: 320px; min-height: 100dvh; overflow-x: clip; background: var(--page-bg); }
```

Rules:

- **Heights**: never `100vh` on mobile (it is the *largest* viewport, content hides under browser bars). Use `100dvh` for full-height pages, `svh` when the element must never jump, and `var(--viewport-height)` for anything that must respect the keyboard.
- **`overflow-x: clip`, not `hidden`**: `hidden` creates a scroll container and breaks `position: sticky` app bars.
- **`overscroll-behavior-y: none`** on the root removes Chrome's pull-to-refresh and the rubber-band showing the page background. Every inner scroller (sheet body, drawer, popover, calendar grid) gets `overscroll-behavior: contain` so scroll never chains to the page.
- **`text-size-adjust: 100%`** stops iOS inflating text in landscape.
- **`scroll-padding-top`** (plus `scroll-margin-top` on anchor targets) keeps focused/anchored elements from landing under the sticky app bar. Update `--app-bar-height` in each breakpoint where the bar changes height.

Safe areas — every edge-touching element pads with `max(<design spacing>, env(safe-area-inset-<side>))`:

```css
.app-bar      { position: sticky; top: 0; min-height: calc(var(--app-bar-height) + env(safe-area-inset-top)); padding-top: env(safe-area-inset-top);
                padding-inline: max(20px, env(safe-area-inset-left)) max(20px, env(safe-area-inset-right)); }
.page         { padding: 42px max(20px, env(safe-area-inset-right)) calc(40px + env(safe-area-inset-bottom)) max(20px, env(safe-area-inset-left)); }
.fab          { position: fixed; right: max(18px, env(safe-area-inset-right)); bottom: max(18px, calc(env(safe-area-inset-bottom) + 10px)); }
.sheet-footer { padding-bottom: calc(14px + env(safe-area-inset-bottom)); }
```

Use `max()` for horizontal insets (notch in landscape replaces the margin) and `calc(+)` for the bottom inset when the element needs breathing room above the home indicator. Use `width: min(<max>, 100%)` + padding instead of `width: calc(100% - 40px)` so the insets can grow the gutter.

Page models:

- **Document pages** (lists, details): body scrolls, sticky app bar.
- **App-shell pages** (calendar, board, map): `height: 100dvh; display: flex; flex-direction: column; overflow: hidden;` with one `flex: 1; min-height: 0` region that scrolls internally (`overscroll-behavior: contain`). This prevents the whole page from bouncing in standalone mode.

Tailwind equivalents: `h-dvh`, `min-h-dvh`, `overscroll-contain`, `pt-[env(safe-area-inset-top)]`, `pb-[max(1rem,env(safe-area-inset-bottom))]`, `max-h-[var(--viewport-height)]`. Keep the `:root` block and the cross-cutting rules in plain CSS: they are global by nature.

### 6. Viewport: keyboard and visual-viewport tracker

`src/shared/pwa/track-viewport-metrics.ts`, called once from the entry file before rendering:

```ts
// iOS never resizes the layout viewport for the on-screen keyboard and Chrome only shrinks the visual one,
// so both are mirrored into CSS: `--viewport-height` (visible area) and `--keyboard-inset` (space the keyboard covers).
// Fixed sheets and toasts read them to stay above the keyboard on Safari and Chrome alike.
const KEYBOARD_THRESHOLD = 120

export function trackViewportMetrics() {
  const viewport = window.visualViewport
  if (!viewport) return
  const root = document.documentElement
  let frame = 0
  let keyboardOpen = false

  function apply() {
    frame = 0
    if (!viewport || Math.abs(viewport.scale - 1) > 0.01) return // pinch zoom, not a keyboard
    const inset = Math.max(0, Math.round(window.innerHeight - viewport.height - viewport.offsetTop))
    const open = inset > KEYBOARD_THRESHOLD
    root.style.setProperty('--viewport-height', `${Math.round(viewport.height)}px`)
    root.style.setProperty('--keyboard-inset', `${inset}px`)
    root.toggleAttribute('data-keyboard-open', open)
    if (open && !keyboardOpen) revealFocusedField()
    keyboardOpen = open
  }

  function schedule() {
    if (!frame) frame = window.requestAnimationFrame(apply)
  }

  viewport.addEventListener('resize', schedule)
  viewport.addEventListener('scroll', schedule)
  apply()
}

// Once a sheet has been lifted above the keyboard, bring the field being typed into back into view.
function revealFocusedField() {
  const field = document.activeElement
  if (!(field instanceof HTMLElement) || !field.matches('input, textarea, select') || !field.closest('dialog[open]')) return
  window.requestAnimationFrame(() => field.scrollIntoView({ block: 'nearest' }))
}
```

Contract exposed to CSS:

| Output | Meaning | Consumers |
| --- | --- | --- |
| `--viewport-height` | Height actually visible (keyboard and browser bars excluded). | `max-height` of sheets, menus, popovers. |
| `--keyboard-inset` | Pixels of the layout viewport covered by the keyboard (0 when closed). | `bottom` of fixed sheets, toasts, any bottom-anchored UI. |
| `html[data-keyboard-open]` | Boolean hook (inset > 120 px, so toolbar collapse is not mistaken for a keyboard). | Drop home-indicator padding, hide grips/FAB while typing. |

Why: on iOS `position: fixed; bottom: 0` sits *behind* the keyboard; on Chrome the layout viewport is unchanged by default, so the same happens. `innerHeight − visualViewport.height − offsetTop` is the covered strip on both engines. The scale guard ignores pinch-zoom; rAF coalesces the burst of resize/scroll events during keyboard animation.

### 7. Interaction model: native touch behaviour

```css
/* Native-app touch model: no double-tap zoom, no long-press callouts, no accidental selection of controls. */
button, a, select, [role='button'], [role='option'], [role='menuitem'] {
  touch-action: manipulation; -webkit-touch-callout: none; -webkit-user-select: none; user-select: none;
}
input, textarea { touch-action: manipulation; -webkit-user-select: text; user-select: text; }
img { -webkit-user-drag: none; }

/* iOS zooms into any focused field under 16px, so touch layouts keep every field at 16px. */
@media (pointer: coarse) {
  input:not([type='checkbox']):not([type='radio']), textarea, select { font-size: 16px !important; }
}

/* iOS Safari only: drop the native date/time chrome that ignores width and centres its value. */
@supports (-webkit-touch-callout: none) {
  input[type='date'], input[type='time'] { -webkit-appearance: none; appearance: none; min-width: 0; }
  input::-webkit-date-and-time-value { min-height: 1.4em; text-align: left; }
}
input[type='search'] { -webkit-appearance: none; appearance: none; }
input[type='search']::-webkit-search-decoration { -webkit-appearance: none; }

button:focus-visible, a:focus-visible, input:focus-visible, textarea:focus-visible, select:focus-visible {
  outline: 3px solid var(--focus-ring); outline-offset: 3px;
}
```

Hover vs press:

- Every `:hover` effect lives inside `@media (hover: hover) { … }`; otherwise touch devices keep the hover state "stuck" after a tap. Tailwind v4's `hover:` variant already does this; hand-written CSS must do it explicitly.
- Give every pressable element an `:active` state that reproduces the hover feedback (translate/shadow/background). For a card whose link is a child: `.card:has(.card__link:active)`.
- iOS only applies `:active` when a touch listener exists on an ancestor. React's root listeners satisfy this; outside React add `document.addEventListener('touchstart', () => {}, { passive: true })`.
- Use `:focus-visible`, not `:focus`, so taps do not leave focus rings.

Field semantics (mobile keyboards):

| Field | Attributes |
| --- | --- |
| Decimal / signed number | `inputMode="decimal"` (keep `type="text"` to allow `+`/`-` and locale separators). |
| Integer | `inputMode="numeric"`, `pattern="[0-9]*"`. |
| Search | `type="search"` + `enterKeyHint="search"`; `autoComplete="off"` for in-app entity search. |
| Email / login / names | `type="email"`, correct `autoComplete` tokens, `autoCapitalize="none"` where relevant. |
| Last field of a form | `enterKeyHint="done"` or `"send"`. |

Text selection: keep it enabled on content (paragraphs, values users may copy); disable it only on controls and on gesture-heavy surfaces (calendars, drag areas) together with `-webkit-touch-callout: none`.

### 8. Overlay system: z-index scale and stacking

Define one scale and use nothing else:

| Layer | z-index | Element |
| --- | --- | --- |
| Sticky app bar | 20 | `.app-bar` (creates a stacking context) |
| Anchored menu / popover | 30 | `.menu__panel` (inside the app bar) |
| FAB | 40 | `.fab` |
| Drawer backdrop | 49 | `.drawer__backdrop` |
| Drawer | 50 | `.drawer` |
| Toasts | 60 | `.toast-stack` |
| Modals / sheets | top layer | `<dialog>` opened with `showModal()` — above every z-index |

Rules:

- Any overlay that must cover the app bar or the FAB is **portaled to `document.body`** (React `createPortal`) or uses the top layer. A child of a stacking context can never escape its parent's z-index.
- Modals are always native `<dialog>` + `showModal()`: top layer, inert background, focus containment, `::backdrop`, `Escape` and the Android back gesture fire `cancel`.
- Freeze the page behind any open overlay: `html:has(dialog[open]), html:has(.drawer) { overflow: hidden; }` (supported on Safari 15.4+/Chrome 105+).
- Toasts sit above the keyboard and above the FAB on phones:

```css
.toast-stack { position: fixed; z-index: 60; right: max(20px, env(safe-area-inset-right));
               bottom: calc(max(20px, env(safe-area-inset-bottom)) + var(--keyboard-inset)); width: min(390px, calc(100vw - 40px)); }
@media (max-width: 760px) {
  .toast-stack { left: max(14px, env(safe-area-inset-left)); width: auto;
                 bottom: calc(max(18px, env(safe-area-inset-bottom) + 10px) + 78px /* FAB height + gap */ + var(--keyboard-inset)); }
}
```

### 9. Overlay system: modals that become keyboard-safe bottom sheets

Structure — header and footer fixed, only the body scrolls:

```html
<dialog class="sheet">
  <div class="sheet__content">
    <span class="sheet__grip" aria-hidden="true"></span>
    <header class="sheet__heading">…</header>
    <div class="sheet__body"><form id="sheet-form">…</form></div>
    <footer class="sheet__footer"><button type="submit" form="sheet-form">Save</button></footer>
  </div>
</dialog>
```

```css
/* Tailwind preflight zeroes the UA `dialog { margin: auto }`, so centring is declared explicitly. */
.sheet { position: fixed; inset: 0; bottom: var(--keyboard-inset); width: min(680px, calc(100vw - 28px)); height: fit-content;
         max-height: min(88dvh, 860px, calc(var(--viewport-height) - 28px)); margin: auto; padding: 0; overflow: hidden; }
.sheet[open] { display: flex; flex-direction: column; }
.sheet::backdrop { background: rgb(40 35 63 / .56); }
.sheet__content { display: flex; flex-direction: column; flex: 1; min-height: 0; }
.sheet__grip { display: none; }
.sheet__heading, .sheet__footer { flex: 0 0 auto; }
.sheet__body { flex: 1; min-height: 0; overflow-y: auto; overscroll-behavior: contain; }
.sheet__footer { padding-bottom: calc(14px + env(safe-area-inset-bottom)); }

@media (max-width: 640px) {
  .sheet { width: 100vw; max-width: none; margin: auto auto 0;
           max-height: min(92dvh, calc(var(--viewport-height) - env(safe-area-inset-top) - 12px));
           animation: sheet-in .22s ease-out; transition: bottom .2s ease-out; }
  .sheet__grip { display: block; }
  .sheet__heading, .sheet__body, .sheet__footer { padding-inline: max(18px, env(safe-area-inset-left)) max(18px, env(safe-area-inset-right)); }
  /* The keyboard already covers the home indicator, so the sheet drops that padding and its grip. */
  html[data-keyboard-open] .sheet__footer { padding-bottom: 12px; }
  html[data-keyboard-open] .sheet__grip { display: none; }
  @keyframes sheet-in { from { transform: translateY(16%); } }
}
```

How it works: `bottom: var(--keyboard-inset)` shrinks the box the dialog is centred/bottom-aligned in, so the sheet rides on top of the keyboard; `max-height` against `--viewport-height` keeps the header reachable; `transition: bottom` follows the keyboard animation; the tracker then scrolls the focused field into view inside `.sheet__body`. Submit buttons live in the footer and target the form with `form="…"`, so actions stay visible while the body scrolls.

Opening without summoning the keyboard — `src/shared/lib/show-modal.ts`:

```ts
// Opens a modal dialog without summoning the on-screen keyboard on touch devices.
// Mouse and keyboard users land on the field marked `data-autofocus`; touch users keep the browser's default focus.
export function showModal(dialog: HTMLDialogElement) {
  if (dialog.open) return
  dialog.showModal()
  if (!window.matchMedia('(pointer: fine)').matches) return
  dialog.querySelector<HTMLElement>('[data-autofocus]')?.focus()
}
```

Rules:

- Replace every `autoFocus`/`autofocus` inside modals with `data-autofocus`; call `showModal(dialog)` from an effect when the open state flips; call `dialog.close()` when it flips back.
- Handle `onCancel={(e) => { e.preventDefault(); if (!busy) onClose() }}` so `Escape`/back gesture go through app state, and close on backdrop tap with `onClick={(e) => { if (e.target === dialogRef.current && !busy) onClose() }}`.
- Do not render heavy content when closed (`{open && …}` inside the dialog) so form state resets per opening.

### 10. Overlay system: drawers

Side drawers (notifications, filters) are not modal dialogs; build them as a portaled pair:

```tsx
{open && createPortal(<>
  <button className="drawer__backdrop" type="button" aria-label="Close" onClick={close} />
  <aside className="drawer" aria-label="…">…</aside>
</>, document.body)}
```

```css
.drawer__backdrop { position: fixed; z-index: 49; inset: 0; border: 0; padding: 0; background: rgb(40 35 63 / .48); }
.drawer { position: fixed; z-index: 50; top: 0; right: 0; display: flex; flex-direction: column; width: min(430px, calc(100vw - 24px)); height: 100dvh; overflow: hidden; }
.drawer__header { padding: max(22px, env(safe-area-inset-top)) max(22px, env(safe-area-inset-right)) 18px 22px; }
.drawer__body { flex: 1; min-height: 0; overflow-y: auto; overscroll-behavior: contain; padding-bottom: calc(22px + env(safe-area-inset-bottom)); }
@media (max-width: 640px) { .drawer { width: 100%; } }
```

Close on backdrop tap, close button and `Escape` (document `keydown` listener registered only while open). The trigger exposes `aria-expanded` and `aria-controls`.

### 11. Overlay system: menus and dropdowns without overlap

Selection controls:

- **Use native `<select>`** for every pick-one control. On iOS/Android it opens the OS picker (wheel/sheet), which is never clipped by `overflow`, never overlaps other UI, respects the keyboard and is accessible. Style only the closed control (`border`, `min-height: 44px`, `font-size: 16px` on touch).
- **Custom autocomplete / search results render in flow**, directly under the input inside the form (a `max-height` list with `overflow-y: auto`), not as an absolutely positioned popover. In-flow lists push content down instead of covering the next field, cannot overlap a sibling dropdown and scroll naturally inside a sheet body.

```css
.search-results { display: grid; max-height: 180px; overflow-y: auto; overscroll-behavior: contain; }
.search-results > button { min-height: 48px; text-align: left; }
```

Anchored menus (account menu, overflow "…" menu):

```css
.menu { position: relative; }
.menu__panel { position: absolute; z-index: 30; top: calc(100% + 9px); right: 0;
               width: min(252px, calc(100vw - 16px));
               max-height: calc(var(--viewport-height) - var(--app-bar-height) - env(safe-area-inset-top) - 16px);
               overflow-y: auto; overscroll-behavior: contain; }
.menu__item { display: flex; align-items: center; width: 100%; min-height: 44px; }
```

Behaviour (React sketch):

```tsx
useEffect(() => {
  if (!open) return
  function onPointerDown(event: PointerEvent) { if (!containerRef.current?.contains(event.target as Node)) setOpen(false) }
  function onKeyDown(event: KeyboardEvent) { if (event.key === 'Escape') setOpen(false) }
  document.addEventListener('pointerdown', onPointerDown)
  document.addEventListener('keydown', onKeyDown)
  return () => { document.removeEventListener('pointerdown', onPointerDown); document.removeEventListener('keydown', onKeyDown) }
}, [open])
```

Rules:

- Close on outside `pointerdown` (not `click`: fires before the next control activates, so two menus are never open at once).
- Anchor to the edge closest to the screen edge (`right: 0` for a right-aligned trigger) and clamp `width` to the viewport.
- Bound `max-height` by `--viewport-height` so the panel never extends behind the keyboard or off-screen.
- Third-party popovers (calendar "+N more", date pickers): clamp `max-width: min(320px, calc(100vw - 24px))` and `max-height: min(320px, 50dvh)` with internal scroll.
- Alternative for new code: the Popover API (`popover` attribute + CSS anchor positioning where supported) gives top-layer rendering and light dismiss natively; keep the same sizing rules.

### 12. Responsive layout for phones, landscape and tablets

- Breakpoints by **behaviour**, not device names. Reference set: `1180px` (shed decorative chrome), `900px` (icon-only account trigger), `760px` (phone layout: smaller app bar, icon-only nav, FAB replaces toolbar "create" button, compact data views), `640px` (modals become bottom sheets, full-width drawers), `430px` (narrowest phones: hide brand text, tighten gaps).
- Phones in landscape: `@media (orientation: landscape) and (max-height: 500px)` → shorter app bar (`--app-bar-height: 56px`), single-row toolbars, hide FAB-dependent duplicates, so the main content keeps usable height.
- Hide labels, not actions: on narrow widths buttons become icon-only and keep an `aria-label`.
- JS-driven layout decisions (e.g. default calendar view on phones) use a `useMediaQuery(query)` hook backed by `matchMedia` + `change` listener, with the same query strings as the CSS.
- Never set fixed widths that exceed `100vw - insets`; use `min(…, 100%)`, `minmax(0, 1fr)` and `min-width: 0` on flex/grid children to prevent horizontal overflow.

### 13. App lifecycle in standalone mode

- **Resume refresh**: standalone apps have no reload button or pull-to-refresh, and iOS does not fire `focus` when relaunching from the home screen. Refresh time-sensitive data on both:

```ts
useEffect(() => {
  const refresh = () => { if (document.visibilityState === 'visible') void reload() }
  // Installed apps resume through visibilitychange; window focus alone misses iOS home-screen launches.
  window.addEventListener('focus', refresh)
  document.addEventListener('visibilitychange', refresh)
  return () => { window.removeEventListener('focus', refresh); document.removeEventListener('visibilitychange', refresh) }
}, [reload])
```

- **Storage and sessions**: on iOS the installed app has its own storage, separate from Safari; users sign in again inside the app. Home-screen apps are exempt from Safari's 7-day script-storage eviction, but optionally call `navigator.storage.persist()`.
- **Authentication**: popup-based OAuth flows are fragile in iOS standalone (popups open in a separate context). Prefer redirect flows or a provider button that works in-page, and test sign-in from the installed app.
- **External links**: links outside `scope` open in an in-app browser sheet (iOS) or Custom Tab (Android); use `target="_blank" rel="noopener"` for them so the app state is preserved.
- **Optional**: detect installed mode with `matchMedia('(display-mode: standalone)')` (plus `navigator.standalone` on older iOS) or style it via `@media (display-mode: standalone)`; offer a custom install button from Chrome's `beforeinstallprompt` and an "Add to Home Screen" hint on iOS.

### 14. Development and verification workflow

- The SW is production-only; test PWA behaviour with a production build (`vite build && vite preview`, or the real container) over HTTPS. For devices, expose the preview through an HTTPS tunnel or a trusted LAN certificate.
- Chrome DevTools → Application: Manifest (errors, icons, maskable preview), Service workers (update on reload, offline toggle), Cache storage.
- Android: `chrome://inspect` remote debugging of the installed app. iOS: Safari → Develop → device → the home-screen app (enable Web Inspector on the device).
- Lighthouse no longer has a PWA category; rely on the Manifest panel and the acceptance matrix below.
- After changing icons on iOS, remove and re-add the app: iOS caches the home-screen icon at install time.

## Files touched (reference layout)

| Path | Purpose |
| --- | --- |
| `index.html` | Head tags (step 1). |
| `public/manifest.webmanifest`, `public/favicon.svg`, `public/favicon-32.png`, `public/icons/*` | Identity (step 2). |
| `public/sw.js` | Service worker (step 3). |
| `src/shared/pwa/register-service-worker.ts` | SW registration. |
| `src/shared/pwa/track-viewport-metrics.ts` | Keyboard/viewport tracker. |
| `src/shared/lib/show-modal.ts` | Keyboard-safe modal opening. |
| `src/shared/hooks/use-media-query.ts` | JS breakpoints. |
| `src/main.tsx` | Calls `trackViewportMetrics()` and `registerServiceWorker()` before `createRoot().render()`. |
| `src/styles/index.css` | `:root` variables, interaction model, overlay layers, sheets, breakpoints. |
| server config (e.g. `nginx.conf`) | Headers (step 4). |

## Anti-patterns

- `100vh`, `bottom: 0` on fixed UI without `--keyboard-inset`, or `window.innerHeight` as "visible height".
- `user-scalable=no` / `maximum-scale=1` to stop focus zoom.
- `autoFocus` inside modals on touch devices.
- Unconditional `:hover` styles; no `:active` feedback.
- Custom absolutely positioned dropdowns inside scrollable forms or sheets; multiple menus able to be open at once.
- Overlays rendered inside sticky/transformed parents with ever-growing z-indexes instead of portals/top layer.
- `overflow-x: hidden` on `body`/`html` (breaks sticky).
- Caching `sw.js`, `manifest.webmanifest` or `index.html`; intercepting `/api/` in the SW; registering the SW in dev.
- `"purpose": "any maskable"`; transparent `apple-touch-icon`.
- Browser/UA sniffing instead of `pointer`, `hover`, `@supports` and `visualViewport`.

## Acceptance criteria

Run on a real iPhone (Safari, installed to home screen), a real Android phone (Chrome, installed) and desktop Chrome.

Identity and delivery

- DevTools Manifest panel shows no errors; Chrome offers install; Android long-press shows the shortcuts.
- Home-screen icon on iOS is opaque with no black corners; Android adaptive icon is not cropped; favicon visible in light and dark browser tabs.
- App opens in standalone (no URL bar); status bar colour matches the app bar; no white flash on launch.
- With network off, the installed app launches to the shell and shows in-app error states for data.
- After a deploy, closing and reopening the app shows the new version without reinstalling.
- `sw.js`, manifest and `index.html` respond with `Cache-Control: no-cache`; hashed assets with `immutable`.

Keyboard

- Tapping a field in a bottom sheet: the sheet rises above the keyboard, the field is visible, the footer actions remain visible; closing the keyboard returns the sheet to the bottom with home-indicator padding.
- Opening a modal by tap does not open the keyboard; on desktop the `data-autofocus` field is focused.
- A toast raised while the keyboard is open is visible above it.
- Focusing any field never zooms the page; pinch zoom still works and does not trigger keyboard logic.

Layers and dropdowns

- Drawer and modals cover the app bar, FAB and page; the page behind does not scroll.
- Every `<select>` opens the OS picker on touch; search results push content down instead of covering it; opening one menu closes any other; outside tap and `Escape` close menus, drawers and modals; Android back gesture closes the open modal.
- No menu or popover extends off-screen or behind the keyboard in portrait or landscape.

Interaction and layout

- No long-press callout or text selection on buttons/links/calendar; no grey tap flash; no double-tap zoom; no stuck hover after tapping; pressed state visible on tap.
- No horizontal scroll at 320 px; content clears notch and home indicator in portrait and landscape; app bar stays sticky.
- No pull-to-refresh or page bounce behind app-shell pages; inner lists scroll without moving the page.
- Returning to the app after it was backgrounded refreshes time-sensitive data.
