# AE MOMENTS

**Trilingual (Hebrew / Russian / English) landing site for an Israeli event-photography brand — instant-print magnets and a premium mirror photo-booth station.**

**Live:** [www.ae-moments.co.il](https://www.ae-moments.co.il)

`Vite 5` · `TypeScript 5.6` · `React 18` · `Tailwind CSS 3.4` · `Framer Motion 11` · `GSAP` · `Vercel` serverless (`fra1`) · `Cloudflare Turnstile` · `Cloudflare Worker` + `D1` · `Vercel KV` · `Resend`

## Serving three languages without three builds

Locale resolves once at boot (`?lang=` beats `localStorage`, which beats the Hebrew default) and is mirrored back into the URL, so the `hreflang` alternates Google indexes resolve to the right content, not a flash of Hebrew. Locale drift is a compile error, not a runtime blank: `TKey = keyof typeof he` with `ru` and `en` typed `Record<TKey, string>`, so a Hebrew string added without its two translations fails `tsc`.

## Hardening the lead form and the browser it runs in

`/api/contact`, outside in: method and body-size gate → Origin allowlist → locked CORS preflight → per-IP rate limit (15 / 10 min, KV-backed) → honeypot → field validation → Turnstile verify → HMAC-SHA256 replay protection fingerprinting each token → HMAC-signed POST to a Cloudflare Worker that inserts into D1 and returns a `lead_id` → Resend delivery → a signed `/status` callback writing the result back onto the row. Ordering matters: Turnstile is verified *before* the replay fingerprint is burned, so a failed CAPTCHA cannot consume a legitimate token, and the handler 500s loudly when `FORM_SECRET` is absent instead of degrading to unsigned. Vercel's Edge Firewall caps it at 30 requests / 60s.

The CSP in `vercel.json` allows `'self'` plus Turnstile, Vercel telemetry and Sentry — no `'unsafe-eval'`, no `'wasm-unsafe-eval'`, no third-party CDNs — beside HSTS `preload` and `X-Frame-Options: DENY`. It had silently regressed to the three.js-era allowances, and was re-tightened only after proving no `eval` or `WebAssembly` consumer remained in the bundle, the widget, or Turnstile.

## Rendering the event clip instead of editing it

`scripts/build-events-video.mjs` builds one self-contained HTML scene where every animation is paused and offset by `animation-delay: calc((START - var(--t)) * 1s)`, so `?t=<sec>` paints an exact frame; headless Chrome screenshots 288 of them and ffmpeg encodes. Deterministic, re-runnable, from the site's own stills. `deband` beat static-grain dither — half the film is flat dark backdrop where grain only adds entropy — taking 9 MB to 1.81 MB, and an 835 K mobile variant took a phone visit from 2.44 MB to 929 K. An earlier pass dropped the three.js hero `Canvas`: ~320 KB gzip for a decorative logo.

## Meeting Israeli Standard 5568 with an embedded widget

Accessibility is the standalone overlay from the `website-accessibility` project, self-hosted as `public/accessibility.js`, targeting Israeli Standard 5568 and WCAG 2.1 AA. It also owns language switching, so the page carries one control. It mounts in an open Shadow DOM: external CSS and `querySelectorAll` cannot reach inside, so `window.__a11yConfig` is the configuration surface.

## How it was verified

- `tests/e2e.mjs` drives Playwright/Chromium against a deployed build as a full-stack smoke — security headers and the contact form's rejection paths exercised against the running stack, not unit mocks.
- The June 2026 hardening loaded the real `dist/` under the *exact* production CSP header in headless Chrome — zero violations. Two adversarial review passes produced 12 and 16 confirmed findings, all applied — including an RTL bug: `inset-inline-end-0` is not a Tailwind v3 utility.
- The apex 307s to `www` and `ae-moments.vercel.app` 308s to `www`, so on `main` the canonical, `hreflang`, OG and all eight JSON-LD blocks in `index.html` point at the host that serves 200, not a redirect. The public `.co.il` host is still pinned to an older deployment pending an owner-side repoint, so what it serves lags `main`.

Source is private.
