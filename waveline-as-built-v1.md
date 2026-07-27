# Waveline Travel — As-Built Record
### What actually shipped, and why it differs from the blueprint

> Companion to `waveline-travel-3d-website-blueprint-v1.md`. The blueprint is the creative direction; this is the implementation record. When the two disagree about *how* something works, this document is correct. When they disagree about *what the site is for*, the blueprint is correct.
>
> Current as of 27 July 2026 (revised after the post-deploy audit — see §7).

---

## 1. Site Inventory

| File | Role |
|---|---|
| `index.html` | Homepage. The only page with the 3D scene. ~96KB self-contained. |
| `about.html` | Advisor bios — Dave and Deanna. ~51KB. |
| `norwegian-cruise-line-travel-advisor.html` | Supplier SEO landing page |
| `carnival-cruise-line-travel-advisor.html` | Supplier SEO landing page |
| `royal-caribbean-travel-advisor.html` | Supplier SEO landing page |
| `disney-cruise-line-travel-advisor.html` | Supplier SEO landing page |
| `disney-resorts-travel-advisor.html` | Supplier SEO landing page |
| `sandals-resorts-travel-advisor.html` | Supplier SEO landing page |
| `sitemap.xml` | 8 URLs, extensionless, each with `<lastmod>` — homepage at priority 1.0, about at 0.8, suppliers at 0.7 |
| `og-image.jpg` | 1200×630 social share card. Referenced by all 8 pages. |
| `favicon.ico` / `favicon.svg` / `apple-touch-icon.png` | Brand mark icons. Linked from all 8 pages. |
| `wrangler.jsonc` | Cloudflare deploy config. Required. See §6. |

The six supplier pages share one template and run ~8.4–8.7KB each: hero → why book through an advisor → what we help you sort out → common questions → CTA.

---

## 2. Why the Stack Diverged

The blueprint recommended Next.js, React Three Fiber, GSAP + ScrollTrigger, and a headless CMS. None of it shipped. The reasoning, recorded so it doesn't get relitigated:

- **A framework had no payer.** Next.js earns its cost through routing complexity, server rendering, or a content pipeline. This site has eight static pages and two people editing them. The framework would have been overhead charged against no benefit.
- **A CMS had no user.** Sanity or Contentful exists so non-technical staff can publish without touching code. There is no non-technical staff. Copy changes go through the same edit → commit → deploy path as everything else.
- **GSAP was solving a problem that turned out to be small.** The blueprint's real requirement was *one scroll-progress value driving both DOM and shader uniforms*. That principle was worth keeping and was kept. It just didn't need a library — a scroll handler and a shared variable did it.
- **React Three Fiber earns its keep when 3D is componentised and reused across routes.** Here the canvas exists on exactly one page and is never remounted. Plain Three.js was the smaller surface.

**The one real cost of this choice:** everything lives in a single file, so there is no component reuse across the eight pages. The nav, footer, and CTA markup are duplicated per page and drift is possible. If a seventh supplier page is ever added, that's the moment to reconsider — not before.

---

## 3. The 3D Scene

Three.js **r128** from CDN. Scoped to the hero and problem sections; the canvas fades out entirely past that point rather than rendering behind content it isn't serving.

**Geometry and shading**
- Two wave planes — near (teal `#1B4F58`) and far (slate-blue `#8FA9B0`) — matching the logo's two-line structure.
- Custom GLSL vertex shader displacing on three summed sine terms at differing frequency and phase, so the swell never visibly loops.
- A sparse glint layer with sine-driven opacity pulsing, plus a very slow rotational drift.
- Horizon glow colour lerps from warm (dawn amber) to steady/white as scroll progresses toward credentials — the blueprint's "light steadies and whitens" beat, implemented as a colour mix on the wave uniforms.

**Camera**
- Starts at `(0, 1.5, 6.2)` looking at `(0, 0.6, -6)`; dollies to `(0, 0.75, 2.1)`.
- Dolly is eased with smoothstep `t*t*(3-2t)` — the "gentle push," not linear.
- **1.5s pre-drift hold.** Camera progress is pinned at 0 until the hold timer completes, regardless of scroll. This is the blueprint's stillness beat and it is load-bearing for the whole opening impression.
- Tilt: on pointer-fine devices, mouse position scaled to ±0.045/0.06 radians. On touch, replaced with an autonomous sine sway an order of magnitude smaller (0.006/0.008). Both smoothed through a 0.05 lerp.

---

## 4. Performance Tiering

The single most consequential engineering decision on the site, and the one that most exceeds what the blueprint specified.

**The problem:** WebGL support is a near-useless signal. Mid-range phones report full support and then render the scene at 12fps, which looks worse than never attempting it.

**The approach:** the static CSS/SVG scene is not a fallback — it's the floor. It paints immediately on first load and remains underneath at all times. The WebGL canvas fades in on top of it *only after earning the right to*, and can be pulled back down at any point.

| Tier | Trigger | Settings |
|---|---|---|
| **Full** | Fine pointer, >1024px | Antialias on, pixel ratio ≤2, cursor-reactive tilt |
| **Lite** | Coarse pointer or ≤1024px | Antialias off, pixel ratio ≤1.5, autonomous sway |
| **Static** | No WebGL, failed benchmark, or demoted | CSS/SVG horizon with keyframe drift |

**Gates, in order:**
1. `prefers-reduced-motion: reduce` → skip the 3D path entirely, never initialise.
2. Initialise on `requestIdleCallback` (2s timeout; 350ms `setTimeout` fallback) so the scene never competes with first paint.
3. `window.THREE` missing → bail silently, stay static. Covers CDN failure.
4. **Warmup:** discard the first 12 frames.
5. **Benchmark:** measure across ~48 frames or 1.4s, whichever comes first. Under **28fps** → demote before the canvas is ever shown.
6. **Watchdog:** ongoing. Frames over 50ms accumulate; frames under it decay the counter at half rate. **3 accumulated seconds** → demote mid-session and stop the render loop.

**Debug overlay:** append `#debug` to any URL. Reports tier (`touch/lite`, `mouse/full`), THREE load status, measured benchmark fps, scene-up confirmation, caught init errors, and demotion events. Not gated behind a build flag — it ships live and is safe to use in production.

---

## 5. Front-End Details

- **Fonts:** Fraunces (`opsz,wght@9..144,500`) for display serif, Inter (400/500/600/700) for body — loaded from Google Fonts, trimmed to only the weights in use.
- **Palette:** teal `#1B4F58`, slate-blue `#8FA9B0`, amber `#E4A032`, warm off-white `#FAF8F4`.
- **Advisor photo:** embedded WebP data URI, converted down from a base64 JPEG. Retained on mobile — an earlier version dropped it and lost the human-trust signal the hero depends on.
- **Easing tokens:** both blueprint eases are present verbatim in the CSS — `cubic-bezier(0.22, 1, 0.36, 1)` for reveals and `cubic-bezier(0.16, 1, 0.3, 1)` for settling.
- **Sticky mobile CTA:** `.sticky-cta`, displayed below 720px, slides up on `translateY`, full-width button at a 48px minimum tap target.
- **Section IDs:** `hero`, `statement`, `problem`, `specialties`, `credentials`, `consultation`.
- **Nav:** Specialties, About (`/about` — extensionless, matching the canonical), Credentials, plus a persistent "Book a consult" button.
- **Backdrop-filter:** solid-colour fallback where unsupported.

---

## 6. Deploy Pipeline

```
edit locally  →  commit & push to GitHub  →  Cloudflare auto-deploys
```

**Cloudflare Workers & Pages**, free tier. Two hard-won constraints:

1. **`wrangler.jsonc` must exist at the repo root** with `"assets": { "directory": "./" }`. Without it the unified Workers & Pages interface fails deploys *silently* — no error, no warning, stale site.
2. **Never use direct-upload mode.** It wipes non-HTML files such as `sitemap.xml` on every deploy. Migrating to GitHub auto-deploy is what fixed this permanently.

**DNS:** managed through Cloudflare. Google Workspace MX and TXT records must be preserved through any DNS change.

**robots.txt:** Cloudflare's Managed robots.txt overrides the repo's custom file. It blocks AI crawlers but not Googlebot. Accepted.

---

## 7. Analytics & SEO

- **GA4:** `G-D4T50YS97X`, linked to Google Search Console.
- **Schema:** homepage carries `TravelAgency` + `Organization` + `Brand`. Supplier pages carry `WebPage` + `WebSite` + `TravelAgency` + `Brand`.
- **Meta:** canonical URL, full Open Graph set (including `og:site_name`, `og:locale`, and `og:image` dimensions/alt), and a `summary_large_image` Twitter card on all eight pages.
- **Icons:** `favicon.ico` + `favicon.svg` + `apple-touch-icon.png`, plus `theme-color: #1B4F58`, on all eight pages.
- **Google Business Profile:** service-area based, Travel Agency category.

### Known gaps

Audited against the live site on 27 July 2026, then remediated the same day.

**Resolved in this pass**

- **`og-image.jpg` was a 404.** All eight pages referenced it; every social share rendered with no preview card. A branded 1200×630 card now ships at the repo root, with `og:image:width`, `og:image:height`, and `og:image:alt` declared so scrapers don't have to fetch the file to lay out the card.
- **No favicon existed.** `favicon.ico`, `favicon.svg`, and `apple-touch-icon.png` are built from the footer brand mark (amber sun, cream wave, teal rounded square) and linked from every page, alongside `theme-color`.
- **Footer social links were dead, and this was a regression.** The handles were wired on 25 July, but the `index.html` in the project folder carried that session's header fix without the social links or the `sameAs` array — both were lost between that deploy and this one. Now restored and live: Facebook, Instagram, X, TikTok, in that order, each `target="_blank" rel="noopener noreferrer me"`. An X icon has been added; the original set only had three. No YouTube account exists yet, so no YouTube icon.
- **`sameAs` restored** on the `TravelAgency` entity in `index.html` and on the `mainEntity` in `about.html`, pointing at all four profiles. This is what tells Google the site and the social accounts are one business.
- **Supplier pages had no Twitter card tags** and **`about.html` was missing `og:site_name` / `og:locale`.** All eight pages now carry an identical social-meta block.
- **Sitemap had no `<lastmod>`.** Added to all eight entries.
- **The six supplier pages didn't link to each other.** Each footer now carries a "We're also certified with" row linking the other five.
- **Supplier CTAs pointed at `/#consultation`,** costing the highest-intent traffic an extra hop. They now go straight to the JourneyFuse request form, with call/email as a secondary line. Disney Resorts uses the Disney-specific form (`pexo4bddsoer`); the rest use the general form (`atlas-f31ee7`).
- **URL form is now consistent.** Canonicals, sitemap, and every internal link use the extensionless form that Cloudflare actually serves.

**Open**

- **The live homepage was serving a stale build.** At audit time `index.html` on the live site still used `.html` internal links while the other seven pages were current — visible text was byte-identical, so this was a deploy or edge-cache issue rather than a content one. Confirm the deployment log and purge cache after publishing this revision.
- **`sitemap.xml` and `robots.txt` could not be verified remotely.** Confirm by eye that the live sitemap is the extensionless version; Search Console may still be following the old `.html` form.
- **No privacy policy page**, while GA4 is collecting. Not a technical fault, but worth a look given client trip data flows through the CRM.
- **GA4 and `FAQPage` schema on the supplier pages could not be confirmed remotely**, because the fetch tooling strips `<script>` contents. Both are confirmed present and valid in source. Verify live via GA4 Realtime and Google's Rich Results Test.

**Structural**

- **Watch for silent regressions across sessions.** The social links and `sameAs` shipped on 25 July and were absent from the file audited on 27 July, while the header fix from the same session survived. Before starting work, diff the project copy against what's actually live rather than assuming the project copy is newer.
- **No shared partials.** Nav, footer, and now the social-meta block are duplicated across eight files. Manageable at current scale; a real liability past ten pages. The social-meta block in particular is now eleven identical lines in eight places.

---

## 8. Decisions Worth Not Relitigating

- **Testimonials stay out** until 2–3 real reviews exist. An empty testimonials section reads worse than no section at all. Slots in as homepage section 5 when ready, with no other layout change needed.
- **Price parity is the trust message.** "Same price as booking direct," no booking fees. This is the answer to the segment's core anxiety and should lead, not hide in the fine print.
- **Cruise first, resorts co-equal, Disney order-taking.** Content weight follows this hierarchy. Disney is captured, not courted.
- **JourneyFuse cannot be iframed.** `myatlasgo.com` sends `X-Frame-Options` blocking cross-origin embedding. Replaced with a direct-contact block; don't re-attempt.
- **Voice is plural.** "We," "your advisors" — never first-person singular, on any page.
