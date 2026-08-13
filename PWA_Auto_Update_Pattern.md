# PWA Auto-Update Pattern

*Canonical reference for shipping an installed PWA that updates itself.*
*Referenced by `Buzz_Project_Development_Procedure.md` §4.1c and the `Project_Context_TEMPLATE.md` setup checklist.*
*Reference implementation: Windex (`windex-expo`), shipped 2026-06.*

---

## The problem this solves

An installed PWA (added to the iOS/Android home screen) caches its JS bundle. With **no service worker**, the OS WebView serves that cached bundle **indefinitely** — a new Vercel deploy does nothing until the user manually deletes and re-adds the app. Two concrete failure modes, both lived through on Windex (2026-06):

1. **Every deploy becomes a manual delete/re-add ritual** for every installed user. For a 20-person app that is unworkable.
2. **Client-side bugs become undebuggable** — you cannot tell whether the user is seeing new code or a stale cached bundle, so you chase ghosts (we burned an afternoon on a "layout bug" that was really a stale bundle masking the real fix).

The fix is two small, always-paired pieces, installed from project setup:

1. **A build-SHA stamp visible in-app** — so you can confirm in two seconds which bundle a device is actually running.
2. **An auto-updating service worker** — network-first on the app shell, registered with the build SHA in its URL, so each deploy auto-updates installed PWAs. Offline still works as a bonus.

> **Order of priority: FRESH CODE FIRST, offline as a bonus — never the reverse.** A badly-scoped SW that caches the shell or bundle *cache-first* recreates exactly the staleness you're trying to kill. Network-first on the HTML and a hashed-by-URL strategy for assets is the whole ballgame.

---

## How it works (the mechanism)

- The SW is registered as **`/sw.js?v=<BUILD_SHA>`**. Each deploy ships a new SHA → a new script URL → the browser sees a "new" service worker → installs it → `skipWaiting()` + `clients.claim()` → fires `controllerchange` in the open page → the page reloads once onto the new bundle.
- The **cache key is the build SHA** (read from the SW's own `?v=`), so `activate()` deletes every previous build's cache.
- **Navigations are network-first**: always fetch live; fall back to the cached shell only when offline (any route resolves to the SPA shell, so offline deep links don't 404).
- The **build SHA is also shown in-app** (a muted footer line), so "is this the new bundle?" is a glance, not a guess.

**Bootstrapping caveat:** the very first SW can't update an app that has no SW yet. So shipping this needs **one** final manual delete/re-add to plant the SW. *Every deploy after that is automatic.*

---

## Invariant core (portable to any PWA, any framework — copy as-is)

1. Build SHA injected at build time **and shown in-app**.
2. SW registered as `/sw.js?v=<BUILD_ID>`.
3. Network-first on navigations (`index.html`) with a **cached-shell fallback for ALL routes**.
4. Cache key = the build SHA; `activate()` purges every other cache.
5. **Cross-origin and non-GET requests are never intercepted** (so Supabase REST/Realtime, POSTs, third-party APIs pass straight through).
6. A documented **self-unregistering kill switch**.
7. **First-activation-skip** — no spurious reload when the SW is first planted.
8. A **reload-loop guard**.
9. Bootstrapping: one final manual re-add plants the SW; everything after auto-updates.
10. **A resume-time update check** — `registration.update()` when the app returns to the foreground, throttled. Registration runs once at mount; an OS that freezes the app and resumes it from a snapshot never re-runs that mount, so without this an install can hold a stale bundle indefinitely. See *iOS freezes installed PWAs* below.

## Per-project adapters (MUST be re-specified each project)

There are exactly **three** things that change per project. They are marked **`CHANGE PER PROJECT (a/b/c)`** in the code below.

- **(a) Static-asset glob for cache-by-URL.** What makes stale-while-revalidate safe is **content-hashed filenames** (the URL changes every build, so a cached entry is always the correct code for that exact URL). Point the glob at the framework's hashed-asset dir:
  - Expo: `/_expo/static/`  ·  Vite: `/assets/`  ·  Next.js: `/_next/static/`  ·  CRA: `/static/`
- **(b) Build-ID env mechanism.** Inject `VERCEL_GIT_COMMIT_SHA` in the build command and expose it to the client via the framework's public-env convention:
  - Expo: `EXPO_PUBLIC_BUILD_ID` (read `process.env.EXPO_PUBLIC_BUILD_ID`)
  - Vite: `VITE_BUILD_ID` (read `import.meta.env.VITE_BUILD_ID`)
  - Next.js: `NEXT_PUBLIC_BUILD_ID` (read `process.env.NEXT_PUBLIC_BUILD_ID`)
- **(c) "Safe to reload now?" hook.** Defer the auto-reload while the user has **unsaved, focused input**, so an update never eats a half-typed message. Windex gates on the chat composer. A project with no such surface uses the **no-op (always-safe)** version — just never call `setComposerBusy(true)`, and reloads happen immediately.

---

## Canonical code

### 1. `public/sw.js` (static file — copied verbatim to `dist/sw.js` / served at `/sw.js`)

```js
/* eslint-env serviceworker, browser */
/*
 * PWA service worker. Strategy — FRESH CODE FIRST, offline as a bonus:
 *   - Navigations (index.html / any SPA route): NETWORK-FIRST, cached-shell
 *     fallback for ANY route when offline (no 404 on offline deep links).
 *   - Immutable hashed assets: STALE-WHILE-REVALIDATE, cached by URL (safe
 *     because the hashed URL changes every build).
 *   - Other same-origin GETs: network-first, cache fallback.
 *   - Cross-origin and non-GET requests: untouched.
 * Cache key = the build SHA from this script's own ?v=, so activate() drops
 * every prior build's cache. See KILL SWITCH below for recovery.
 *
 * ── KILL SWITCH (recovery if this SW ever misbehaves) ──────────────────────
 *   1. Replace this ENTIRE file's body with the block below.
 *   2. Commit + push (the build redeploys the PWA).
 *   3. Installed apps register /sw.js?v=<new SHA>, fetch this self-destructing
 *      script, wipe all caches, unregister the SW, and reload to a clean,
 *      SW-less state (plain network — always safe).
 *   4. Once propagated, restore this file (or ship a corrected SW) and deploy.
 *
 *   self.addEventListener('install', (e) => {
 *     e.waitUntil((async () => {
 *       const keys = await caches.keys();
 *       await Promise.all(keys.map((k) => caches.delete(k)));
 *       await self.skipWaiting();
 *     })());
 *   });
 *   self.addEventListener('activate', (e) => {
 *     e.waitUntil((async () => {
 *       await self.registration.unregister();
 *       const cs = await self.clients.matchAll({ type: 'window' });
 *       cs.forEach((c) => c.navigate(c.url));
 *     })());
 *   });
 * ───────────────────────────────────────────────────────────────────────────
 */

const BUILD = new URLSearchParams(self.location.search).get('v') || 'dev';
const CACHE = `app-${BUILD}`;

self.addEventListener('install', (event) => {
  event.waitUntil(
    (async () => {
      try {
        const cache = await caches.open(CACHE);
        await cache.add('/index.html'); // precache the shell for offline
      } catch (err) {
        /* best-effort; never fail install (that blocks activation) */
      }
      await self.skipWaiting();
    })()
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      const keys = await caches.keys();
      await Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k)));
      await self.clients.claim();
    })()
  );
});

// ⚠️ CHANGE PER PROJECT (a): point this at the framework's content-hashed
// asset directory. Expo: '/_expo/static/' · Vite: '/assets/' ·
// Next.js: '/_next/static/' · CRA: '/static/'.
function isHashedAsset(url) {
  return url.pathname.startsWith('/_expo/static/');
}

self.addEventListener('fetch', (event) => {
  const request = event.request;
  if (request.method !== 'GET') return; // never touch POST/PUT/etc.

  const url = new URL(request.url);
  if (url.origin !== self.location.origin) return; // leave cross-origin alone

  // SPA navigations: network-first, fall back to the cached shell for ANY route.
  if (request.mode === 'navigate') {
    event.respondWith(
      (async () => {
        const cache = await caches.open(CACHE);
        try {
          const fresh = await fetch(request);
          cache.put('/index.html', fresh.clone()); // the rewrite serves the shell
          return fresh;
        } catch (err) {
          const shell = await cache.match('/index.html');
          return shell || Response.error();
        }
      })()
    );
    return;
  }

  // Immutable hashed assets: stale-while-revalidate by URL.
  if (isHashedAsset(url)) {
    event.respondWith(
      (async () => {
        const cache = await caches.open(CACHE);
        const cached = await cache.match(request);
        const network = fetch(request)
          .then((resp) => {
            if (resp && resp.ok) cache.put(request, resp.clone());
            return resp;
          })
          .catch(() => cached);
        return cached || network;
      })()
    );
    return;
  }

  // Other same-origin GETs: network-first, cache as offline fallback.
  event.respondWith(
    (async () => {
      const cache = await caches.open(CACHE);
      try {
        const fresh = await fetch(request);
        if (fresh && fresh.ok) cache.put(request, fresh.clone());
        return fresh;
      } catch (err) {
        const cached = await cache.match(request);
        if (cached) return cached;
        throw err;
      }
    })()
  );
});
```

### 2. Registration + composer-aware reload (`lib/pwaUpdate.ts`)

```ts
import { Platform } from 'react-native'; // (RN/Expo) — on web-only frameworks, drop this and the Platform guard

let composerBusy = false; // user has unsaved, focused input
let reloadPending = false;
let refreshing = false; // a reload is already in flight (in-memory loop guard)

// Resume-time update checks: at most one per minute, so rapid app switching
// doesn't fire a burst of sw.js requests.
let lastUpdateCheck = 0;
const UPDATE_CHECK_MIN_INTERVAL_MS = 60_000;

function maybeReload(): void {
  if (refreshing || !reloadPending || composerBusy) return;
  if (typeof window === 'undefined') return;
  // Cross-reload loop guard: never reload twice within 10s in one tab session.
  try {
    const KEY = 'app-sw-last-reload';
    const last = Number(window.sessionStorage.getItem(KEY) || '0');
    if (Date.now() - last < 10000) return;
    window.sessionStorage.setItem(KEY, String(Date.now()));
  } catch {
    /* sessionStorage unavailable — rely on the in-memory refreshing guard */
  }
  refreshing = true;
  reloadPending = false;
  window.location.reload();
}

// ⚠️ CHANGE PER PROJECT (c): the "safe to reload now?" hook. Call this from
// the surface that holds unsaved focused input (Windex: the chat composer).
// Projects with no such surface never call it with `true` → reloads are
// always immediate (no-op always-safe).
export function setComposerBusy(busy: boolean): void {
  composerBusy = busy;
  if (!busy) maybeReload();
}

export function registerServiceWorker(buildId: string): void {
  if (Platform.OS !== 'web' || typeof navigator === 'undefined' || !('serviceWorker' in navigator)) {
    return;
  }
  // First-activation-skip: if there's no controller yet, the first activation
  // is the INITIAL install, not an update — don't reload for that one.
  let sawController = !!navigator.serviceWorker.controller;
  navigator.serviceWorker.addEventListener('controllerchange', () => {
    if (!sawController) {
      sawController = true;
      return;
    }
    reloadPending = true;
    maybeReload();
  });
  navigator.serviceWorker
    .register(`/sw.js?v=${buildId}`)
    .then((registration) => {
      // This register() call runs ONCE, at mount. iOS freezes an installed
      // standalone PWA and resumes it from a snapshot WITHOUT re-navigating, so
      // on a home-screen install that mount may not happen again for days — and
      // the app keeps running whatever bundle it was frozen with. Re-check on
      // every foreground; if a new SW is found, the controllerchange path above
      // applies it (composer-aware, loop-guarded).
      if (typeof document === 'undefined') return;
      document.addEventListener('visibilitychange', () => {
        if (document.visibilityState !== 'visible') return;
        const now = Date.now();
        if (now - lastUpdateCheck < UPDATE_CHECK_MIN_INTERVAL_MS) return;
        lastUpdateCheck = now;
        registration.update().catch(() => {
          /* offline or transient — the next resume tries again */
        });
      });
    })
    .catch(() => {
      /* registration failure is non-fatal; the app still works without the SW */
    });
}
```

### 3. Register on app start (web-guarded), e.g. `app/_layout.tsx`

```ts
useEffect(() => {
  // ⚠️ CHANGE PER PROJECT (b): the env var name (see vercel.json below).
  registerServiceWorker((process.env.EXPO_PUBLIC_BUILD_ID ?? '') || 'dev');
}, []);
```

### 4. Inject the build SHA at build time — `vercel.json`

```json
{
  "buildCommand": "EXPO_PUBLIC_BUILD_ID=$VERCEL_GIT_COMMIT_SHA npx expo export --platform web"
}
```

⚠️ **CHANGE PER PROJECT (b):** the env var name and build command per framework.
- Vite: `"buildCommand": "VITE_BUILD_ID=$VERCEL_GIT_COMMIT_SHA vite build"` → read `import.meta.env.VITE_BUILD_ID`.
- Next.js: set `NEXT_PUBLIC_BUILD_ID=$VERCEL_GIT_COMMIT_SHA` in the build command (or `next.config` env) → read `process.env.NEXT_PUBLIC_BUILD_ID`.

### 5. Show the SHA in-app (the stamp)

A muted, low-contrast line on a low-traffic surface (Windex: the drawer footer). Trimmed to a short SHA; `'dev'` for local builds.

```tsx
const BUILD_ID = (process.env.EXPO_PUBLIC_BUILD_ID ?? '').slice(0, 7) || 'dev';
// ...render <Text style={muted}>build {BUILD_ID}</Text>
```

---

## Kill-switch deploy procedure (verbatim)

If the SW ever misbehaves (caches wrong, won't update, serves broken code):

1. Replace the **entire body** of `public/sw.js` with the self-destructing block from its header comment (wipes all caches, `unregister()`s, reloads clients).
2. Commit + push to the deployed branch (the PWA rebuilds).
3. Installed apps register `/sw.js?v=<new SHA>`, fetch the self-destructing script, **wipe all caches**, **unregister** the SW, and reload to a clean, SW-less state (plain network — always safe).
4. Once it has propagated, restore `sw.js` (or ship a corrected SW) and deploy.

The full `install`/`activate` kill code lives as a copy-paste block in the `sw.js` header so it never has to be reconstructed.

---

## Verification ritual (proves auto-update on any project)

1. **Ship** the SW + stamp.
2. Do **one** final manual delete/re-add of the installed PWA to plant the SW. Open the in-app stamp and note the SHA.
3. Push **any trivial change** (a one-line comment) → new deploy → new SHA.
4. **Without re-adding**, reopen the installed PWA. The in-app build SHA should **flip to the new commit on its own**.

If the SHA flips with no re-add, auto-update works and the manual ritual is dead for good.

---

## Related caveat — iOS freezes installed PWAs, so registration-at-mount is not enough

`registerServiceWorker` is called from an app-start effect. On iOS, a home-screen install is **frozen and resumed from a snapshot without re-navigating**, so that effect can go days without running again. Everything in the mechanism above hangs off it — meaning an installed app can sit on a stale bundle indefinitely while every other device updates normally. Invariant 10 (the `visibilitychange` → `registration.update()` check) is what closes this.

Two consequences worth internalizing:

- **A frozen install cannot deliver its own fix.** Shipping the resume check does nothing for an app already frozen without it — the very mechanism needed to pick it up is the thing being shipped. Those users need **one** force-close-and-reopen (or one load in the browser). Budget for that when you add invariant 10 to an existing project, exactly as you budget the one manual re-add that plants the SW initially.
- **"Is the user on new code?" is not answerable from the deploy.** A green deploy plus a flipped bundle hash proves the *server* changed. Only the in-app build stamp on that device proves the *client* did. This is the same lesson as the build-SHA stamp, but it bites hardest on iOS installs.

*Origin: Windex, 2026-08-12. A player's sign-in failed silently in an installed PWA; the login screen's own gaps (no request timeout, no catch, feedback rendered below the fold) hid it, and a frozen install was the most consistent explanation for a request that never reached the server at all. Fixed in `e253e50`.*

---

## Related caveat — a standalone install and the browser have SEPARATE storage

On iOS, a home-screen install and Safari are **different storage contexts**. `localStorage`, cookies, and therefore any persisted auth session are not shared. Signing in inside Safari leaves the installed app **still signed out**, and vice versa.

This is expected platform behavior, not a bug — but it reliably reads as one, to users and to whoever is debugging:

- A user who signs in via an emailed link or code (which opens in **Safari**), then opens the home-screen icon, finds themselves at the login screen and reasonably concludes sign-in "didn't work."
- Single-use codes make it worse: the code was consumed by the Safari sign-in, so re-entering it in the installed app fails as already-used — which looks like an expiry bug and sends you investigating token lifetimes that are working perfectly.

**How to apply:** for any invite/OTP flow on an installable PWA, expect every invitee to hit this once, and say so in the onboarding copy ("if you added Windex to your home screen, sign in there"). When triaging "my code didn't work," establish **which context** the user was in before touching tokens or templates — the request's `User-Agent` distinguishes them (a standalone install omits `Version/<n>` and `Safari/<n>`, which mobile Safari includes).

*Origin: Windex, 2026-08-12. A player signed in successfully in Safari at 01:34Z, added the app to his home screen, and was met with a login screen; his reuse of the already-consumed code then failed as expired. Both behaviors were correct, and neither was what he'd been told to expect.*

---

## Related caveat — iOS input auto-zoom (masquerades as a layout bug)

iOS Safari (and standalone PWAs) **auto-zoom when a focused input has `font-size < 16px`**. The zoom makes the page wider than the frame; iOS then scrolls to keep the input visible, which shoves left-anchored UI (send buttons, tab bars) **off-screen**. It looks exactly like a CSS/layout bug and will send you chasing transforms and viewport math.

**Fix: every focusable input has `font-size >= 16px`.** 16 is the exact threshold.

*Origin: Windex chat composer, 2026-06. We instrumented `visualViewport` on-device and proved the displacement was the auto-zoom (window dimensions shrank ~15/16 on focus, with zero horizontal viewport offset), not any offset math. One-line fix: input `fontSize: 15 → 16`.*
