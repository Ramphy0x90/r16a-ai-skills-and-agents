# AI crawler reference

Last verified: October 2026. This list changes — if it's more than ~6 months old, re-check each operator's official docs before relying on it.

| Operator | robots.txt token | Purpose | Block it if… |
|---|---|---|---|
| OpenAI | `GPTBot` | Model training | you don't want content used for training |
| OpenAI | `OAI-SearchBot` | ChatGPT search index | you want to disappear from ChatGPT search (usually: keep allowed) |
| OpenAI | `ChatGPT-User` | Fetch triggered by a user's request | rarely; robots.txt may not apply, enforce at CDN/WAF |
| Anthropic | `ClaudeBot` | Model training | you don't want content used for training |
| Anthropic | `Claude-SearchBot` | Claude search index | you want to disappear from Claude search (usually: keep allowed) |
| Anthropic | `Claude-User` | Fetch triggered by a user's request | rarely |
| Perplexity | `PerplexityBot` | Perplexity search index | you want to disappear from Perplexity |
| Perplexity | `Perplexity-User` | User-triggered fetch | rarely; reported to ignore robots.txt, enforce at CDN/WAF |
| Google | `Googlebot` | Google Search (incl. AI Overviews / AI Mode) | never, for a public site |
| Google | `Google-Extended` | **Control token, not a crawler.** Opts out of Gemini training/grounding use. Doesn't affect Google Search or AI Overviews eligibility | you don't want Gemini training/grounding |
| Apple | `Applebot` | Siri/Spotlight/Apple search | rarely |
| Apple | `Applebot-Extended` | **Control token.** Opts out of Apple AI training only | you don't want Apple training use |
| Meta | `meta-externalagent` | Training | you don't want training use |
| Common Crawl | `CCBot` | Open dataset widely used for training | you don't want training use |

Control tokens (`Google-Extended`, `Applebot-Extended`) never show up as their own user agent in server logs.

## Templates

**Visible in AI search, opted out of training** (common choice for a product site):

```
User-agent: GPTBot
Disallow: /

User-agent: ClaudeBot
Disallow: /

User-agent: Google-Extended
Disallow: /

User-agent: Applebot-Extended
Disallow: /

User-agent: meta-externalagent
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: *
Allow: /
Disallow: /app/
Disallow: /api/

Sitemap: https://example.com/sitemap.xml
```

**Allow everything** (maximum reach, e.g. docs or open-source project):

```
User-agent: *
Allow: /
Disallow: /app/
Disallow: /api/

Sitemap: https://example.com/sitemap.xml
```

Adjust `/app/` and `/api/` to the real authenticated and API paths. Always confirm the owner's training-vs-search choice before writing either.

Sources to re-verify against: OpenAI's bots page (platform.openai.com/docs/bots), Anthropic's crawler support article (support.claude.com), Google's crawler list (developers.google.com/search/docs/crawling-indexing/overview-google-crawlers), Perplexity's bots page (docs.perplexity.ai).
