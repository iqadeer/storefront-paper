---
name: dev-local
description: Local dev gotchas — real-device testing via ngrok/LAN (ALLOWED_DEV_ORIGINS), Chrome-iOS hydration noise, and the Next 16 image-optimizer SSRF guard that 400s /_next/image for localhost Saleor media (dev-only dangerouslyAllowLocalIP). Use when client components seem broken over a tunnel or on a phone, or when images show placeholders in local dev.
---

# Local Development & Mobile Testing

Patterns for running `pnpm dev` on a real phone (ngrok, LAN IP, tunnel) without mistaking Next.js dev restrictions for product bugs.

---

## Cross-origin dev resources (ngrok / tunnels)

### Symptom

Testing on a **real device** via ngrok (or similar) while the dev server runs on your laptop:

- Client components behave broken (carousels won't swipe, buttons feel dead, hydration oddities)
- Browser console shows Next.js blocking `/_next/*` (HMR, chunks, dev middleware)
- **Chrome DevTools responsive mode on localhost still works** — the page origin matches the dev server

This is **not** a production bug and often **not** a component/touch bug. The HTML loads from the tunnel host, but dev assets are rejected when the browser treats the request as cross-origin.

### Fix

Allow the tunnel hostname in `allowedDevOrigins` and **restart** `pnpm dev`.

Paper reads hostnames from `.env.local`:

```env
# Hostname only — no https://, comma-separated for multiple tunnels
ALLOWED_DEV_ORIGINS=servilely-quare-polly.ngrok-free.dev
```

`next.config.js` maps that to Next.js:

```javascript
allowedDevOrigins: ["servilely-quare-polly.ngrok-free.dev"];
```

After changing `ALLOWED_DEV_ORIGINS` or `next.config.js`, restart the dev server.

### LAN testing (`--hostname 0.0.0.0`)

Same rule applies when you open `http://192.168.x.x:3000` from your phone. Add the IP (or a stable local hostname) to `ALLOWED_DEV_ORIGINS` if dev chunks are blocked.

### When to use production build instead

For final QA of touch/gesture behavior without dev middleware:

```bash
pnpm build && pnpm start
```

Tunnel or LAN to the production server — no `allowedDevOrigins` needed.

---

## Chrome on iOS: `__gcruniqueid` hydration warnings

### Symptom

After ngrok/LAN dev works, the console shows a hydration mismatch on `<form>` / `<input>` (often `SearchBar` in the header):

```diff
  <form ...>
-   __gcruniqueid="1"
  <input ...>
-   __gcruniqueid="2"
```

Paper does **not** render these attributes — grep the repo finds nothing.

### Cause

**Chrome (and Chromium-based Edge) on iOS** inject `__gcruniqueid` / `__gchrome_uniqueid` on form fields for autofill **after** the server HTML is sent but **before** React hydrates. React then warns because client DOM ≠ server HTML.

This is a [known Chromium + React limitation](https://github.com/vercel/next.js/issues/77710). Safari on iOS typically does not inject these attributes.

### What to do

| Goal                | Action                                                                                                     |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| Confirm it's benign | Reproduce in **Safari** on the same phone — warning usually disappears                                     |
| Ignore in dev       | Safe — **no user-visible breakage** in production per Next.js/React guidance                               |
| Quieter dev console | Avoid Chrome on iOS for day-to-day mobile QA; use Safari                                                   |
| Last resort         | `suppressHydrationWarning` on affected inputs — silences real mismatches too; not recommended project-wide |

Do **not** refactor `SearchBar` or disable SSR on the header to silence this.

---

## Debugging checklist (mobile-only issues)

1. **Console on the phone** — Safari Web Inspector (Mac) or Eruda; look for Next.js "Cross-origin access to Next.js dev resources" first.
2. **`ALLOWED_DEV_ORIGINS`** — tunnel hostname listed? Dev server restarted?
3. **Origin mismatch** — ngrok URL in the address bar must match the hostname in `ALLOWED_DEV_ORIGINS` (subdomain changes when ngrok restarts free tunnels).
4. **Component layer** — only after dev origins are clean; e.g. Embla carousels need `touch-pan-y` on the viewport for real iOS touch (see `src/ui/components/ui/carousel.tsx`).
5. **Hydration on `__gcruniqueid`** — Chrome on iOS autofill injection; not a storefront bug (see above).

---

## Image optimizer 400s on localhost media (Next 16 SSRF guard)

### Symptom

- Catalog pages (home, PLP, PDP, category) show images fine, but **cart drawer, cart page, and checkout order summary show placeholders** in local dev.
- Dev log repeats: `⚡ upstream image http://localhost:8001/... hostname resolved to private IP ["::1","127.0.0.1"] If this is expected and you understand SSRF risk, use images.dangerouslyAllowLocalIP = true to continue.`
- `/_next/image?url=http%3A%2F%2Flocalhost%3A8001%2F…` responds **400**.

### Cause

Two image pipelines: `SaleorImage` with a Saleor `srcset` (`buildSaleorSrcSet`, default `saleor` pipeline in `src/lib/images.ts`) renders a plain `<img>` pointing straight at Saleor — never touches the optimizer, works with localhost. Cart drawer / cart page / order summary use raw `next/image` (no srcset) → `/_next/image?url=…`.

Next 16's optimizer has an SSRF guard that **refuses upstreams resolving to private IPs** (127.0.0.1/::1). It runs **after** the `remotePatterns` allowlist — so the dev wildcard `hostname: "*"` in `next.config.js` does not help. Production with a public API host is unaffected.

### Dev fix (already in place — keep it)

`next.config.js` → `images`:

```javascript
...(isDevelopment ? { dangerouslyAllowLocalIP: true } : {}),
```

- Changing the local Saleor port needs **no change here** — the guard is IP-class based, not URL/port based; any `localhost:<port>` upstream passes in dev.
- `next.config.js` changes require a dev-server **restart** (`pnpm dev`).
- Verify: `curl -s -o /dev/null -w "%{http_code} %{content_type}\n" "http://localhost:3000/_next/image?url=http%3A%2F%2Flocalhost%3A8001%2Fthumbnail%2FUHJvZHVjdE1lZGlhOjM2%2F256%2Fwebp%2F&w=128&q=75"` → `200 image/webp`.

### Production (revert / verify)

- **Nothing to revert while the spread stays gated on `isDevelopment`** — a production build never includes the flag, so the SSRF guard stays active.
- Never un-gate it or set `dangerouslyAllowLocalIP` unconditionally: it would let any request use the storefront as a proxy into private networks.
- Pre-deploy check: `grep -n dangerouslyAllowLocalIP next.config.js` — only the dev-gated line may remain.
- Code-level alternative (no config, for new surfaces): render these images with `SaleorImage` + a thumbnail `srcset` like the catalog pages do.

---

## Anti-patterns

❌ **Don't debug carousel swipe on ngrok** before fixing `allowedDevOrigins` — client JS may not load  
❌ **Don't commit personal ngrok hostnames** — use `ALLOWED_DEV_ORIGINS` in `.env.local`  
❌ **Don't assume Chrome device toolbar = real phone** — it uses localhost + mouse events, not tunnel + touch
❌ **Don't widen `remotePatterns` to fix localhost image 400s** — the SSRF guard runs after the allowlist; use the dev-gated `dangerouslyAllowLocalIP` (see above)
❌ **Don't enable `dangerouslyAllowLocalIP` unconditionally** — dev-gated only; it is an SSRF bypass
