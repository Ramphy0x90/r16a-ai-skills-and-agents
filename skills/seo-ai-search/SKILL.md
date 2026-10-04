---
name: seo-ai-search
description: Use when building or changing public web pages, before launching a site, or when auditing SEO and AI-search visibility (Google, AI Overviews, ChatGPT, Claude, Perplexity). Covers rendering, robots/sitemap, metadata, structured data, CWV.
---

# SEO & AI-search visibility

Search engines and AI search tools can only use what they receive in the HTTP response. Most damage comes from a few sitewide mistakes (content not in the HTML, a blocking robots rule, `noindex` left on), not from missing polish. Check sections 1–2 first.

Only public, indexable pages matter. Anything behind login should be `noindex` or disallowed, and otherwise left alone.

## 1. Rendering & crawlability

- **Main content must be in the server-delivered HTML.** Google renders JavaScript, but with a delay. Most AI crawlers (GPTBot, ClaudeBot, PerplexityBot) don't execute JavaScript at all. A client-only SPA shows them an empty shell. Check with `curl -sL <url>` and look for the page's headline and body text.
- Per stack:
  - **Angular**: SSR or prerendering via `@angular/ssr` (`ng add @angular/ssr`). Marketing pages should be prerendered.
  - **Next/Nuxt/SvelteKit/Astro**: SSR/SSG is the default. Check that key pages aren't switched to client-only rendering.
  - **Flutter web** renders to a canvas, with little semantic HTML. Don't rely on it for pages that need to rank. Put landing/docs pages on a static or SSR site and keep Flutter for the app.
- Real URLs per page (path-based routing, not `#/` hash routes). Internal links are `<a href>`, not click handlers.
- Correct status codes: real 404s for missing pages (not 200 with a "not found" view, a "soft 404"), 301 for permanent moves, no redirect chains.
- One canonical host. `http`→`https` and `www`/apex should redirect to a single version.
- No accidental `noindex`: check `<meta name="robots">`, the `X-Robots-Tag` header, and framework-level SEO settings. Staging/preview environments **should** be `noindex` or behind auth. Check that this doesn't leak into prod config, and that prod doesn't leak into staging.

## 2. robots.txt, sitemap & AI crawlers

- `robots.txt` at the root. `Disallow: /` under `User-agent: *` blocks everything. That's critical on a public site.
- Don't disallow CSS/JS needed to render the page.
- `Disallow` stops crawling, not indexing. To keep a page out of results, use `noindex`, and leave the page crawlable so the tag can be seen.
- List the sitemap in `robots.txt` (`Sitemap: https://…/sitemap.xml`). The sitemap should contain only canonical, indexable, 200-status URLs with absolute URLs. Generate it from the routes rather than hand-maintaining it.
- **AI crawlers:**
  - **Training vs. search:** blocking a training crawler (`GPTBot`, `ClaudeBot`, `CCBot`, `Google-Extended`) doesn't remove the site from AI search. Blocking a search crawler (`OAI-SearchBot`, `Claude-SearchBot`, `PerplexityBot`) does.
  - **Tokens and templates:** see `${CLAUDE_SKILL_DIR}/references/ai-crawlers.md`. If that path doesn't resolve, Glob for `**/seo-ai-search/references/ai-crawlers.md` under `~/.claude` or `.claude`.
  - **Confirm the choice:** the training-vs-search choice belongs to the owner. Confirm it before writing rules.
- A CDN/WAF bot-protection setting can return 403 to AI user agents even when `robots.txt` allows them. Check the actual responses.

## 3. Page metadata

- **Title:** a unique `<title>` per page, roughly 50–60 characters, with the specific topic first and the brand last. In SPAs/SSR apps, set it per route (Angular `Title`/`Meta` services or route `title`; Next.js `metadata`) — don't leave one static title in `index.html`.
- **Description:** a unique meta description per page, about 150–160 characters. It doesn't affect ranking but drives the snippet.
- **Canonical:** `<link rel="canonical">` with an absolute URL on every indexable page, self-referencing unless the page is a duplicate. Query-string variants (sorting, tracking params) should point to the clean URL.
- **Headings:** one `<h1>` matching the page's topic, and a logical `h2`/`h3` structure.
- **Images:** descriptive `alt` on meaningful images.
- **Language:** `lang` on `<html>`. For multi-language sites, `hreflang` alternates that reference each other, plus `x-default`.
- **Social:** Open Graph (`og:title`, `og:description`, `og:image` absolute URL, `og:url`) and `twitter:card` on shareable pages. These are rendered server-side — social scrapers don't run JS.

## 4. Structured data

- JSON-LD (`<script type="application/ld+json">`) in the server HTML, using schema.org types that fit the page: `Organization` + `WebSite` on the home page, `Article`/`BlogPosting`, `Product` with `Offer`, `SoftwareApplication`, `BreadcrumbList`, `FAQPage` only for real Q&A content.
- The data must match visible content. Marking up things not on the page (fake reviews/ratings, hidden FAQs) violates Google's guidelines and can cost rich results.
- Validate with Google's Rich Results Test or the schema.org validator when a URL is available. Note that Google shows rich results for only a subset of types.

## 5. Performance (Core Web Vitals)

Google's "good" thresholds, measured at the 75th percentile:
- **LCP ≤ 2.5 s:** don't lazy-load the hero image. Give it `fetchpriority="high"`, serve modern formats at the right size, and avoid render-blocking fonts/CSS.
- **INP ≤ 200 ms:** avoid long main-thread tasks on interaction. Watch heavy hydration and large third-party scripts.
- **CLS ≤ 0.1:** set `width`/`height` (or `aspect-ratio`) on images, embeds and ads. Use `font-display` plus size-matched fallbacks. Don't inject banners above existing content.

When a URL is available, PageSpeed Insights / Lighthouse gives lab data. Field data (CrUX) is what ranking uses.

## 6. Content AI search can cite

- Answer the page's core question plainly near the top, in text. Not only in images, video, canvas, or behind tabs/accordions that load content on click.
- Use specific, verifiable facts (numbers, names, dates, versions) and descriptive headings that match how people ask. AI answers quote passages that stand on their own.
- Keep important facts consistent across pages and with structured data. Show a visible last-updated date where freshness matters.
- One topic per URL. Thin or near-duplicate pages (e.g. one per city with swapped names) hurt both classic and AI search.
- `llms.txt` is optional. Major AI search crawlers aren't documented as using it, so don't present it as a ranking factor.
- Don't promise rankings or AI citations. Neither can be guaranteed.

## How to run this

1. Identify the stack and rendering mode, and list the public routes.
2. Walk sections 1–6 against the code. If a URL is available, also check the live responses (`curl -sL -A "GPTBot" <url>`, `curl -sI <url>`, `/robots.txt`, `/sitemap.xml`).
3. Fix what's straightforward (missing per-route titles/canonicals, sitemap entries, image dimensions). Flag decisions instead of making them: the AI-training opt-out, moving a site to SSR, URL structure changes.
4. If the change touched no public page, say so instead of forcing findings.
