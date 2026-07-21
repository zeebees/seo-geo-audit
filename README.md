# seo-geo-audit

An [Agent Skill](https://agentskills.io/specification) that audits a website for
technical SEO **and** AI-answer-engine discoverability (GEO) — whether ChatGPT,
Perplexity, Gemini, Claude and Google can crawl, understand, and cite it.

Output is a single evidence-bearing table. Every row carries the observed bytes.

## Why this exists

Coding agents already know general SEO. Ask one to audit a site and it will find
your canonical tags, your `hreflang`, your missing `sameAs`. This skill is
deliberately **not** that. It covers the two things an unaided agent reliably
misses:

**1. Checks that require fetching the site a specific way.** Markup lies. Reading
a `robots.txt` that allows `PerplexityBot` is not evidence that `PerplexityBot`
can fetch the site — many CDN and WAF products ship an AI-bot block that
`robots.txt` cannot override. So the skill fetches *as* each answer crawler,
diffs bot HTML against human HTML to catch cloaking, tests the bare apex
separately from `www`, and checks a second fetch for a cache HIT.

**2. Facts that changed recently.** Google retired FAQ rich results entirely in
May 2026. GA4 started attributing AI referral traffic on 13 May 2026.
`Google-Extended` is a control token, not a crawler. IndexNow does not reach
Google. An agent working from older training data gets these wrong with total
confidence.

Every check is tagged `[proven]`, `[bet]`, or `[contested]` so the report
distinguishes documented behavior from a cheap hunch — and the skill is
instructed not to promote a bet into a recommendation to sound more decisive.

## Install

Copy the skill into your agent's skills directory:

```bash
git clone https://github.com/USER/seo-geo-audit
cp -r seo-geo-audit/skills/seo-geo-audit ~/.claude/skills/
```

Then just ask: *"audit example.com for SEO and AI discoverability"*.

## What it checks

| Group | Covers |
|---|---|
| **A. Hostname & canonical** | apex vs `www` both resolving, one-200 rule, self-referential canonicals, slash/query variants, HTTP→HTTPS |
| **B. Languages** | one URL per language (catches same-URL JS toggles), reciprocal `hreflang`, `x-default`, no phantom translations |
| **C. Page basics** | single `<h1>`, unique title/description, semantic landmarks, heading anchors, server rendering |
| **D. Structured data** | `BlogPosting`/`BreadcrumbList`/`WebSite`, `</` escaping, E-E-A-T authorship, `FAQPage`, `sameAs` |
| **E. AI crawler access** | answer vs user-triggered vs training crawlers, live per-bot fetch test, `noindex` on AI-facing copies, `.md` alternates, `llms.txt`, IndexNow |
| **F. Speed & caching** | edge-cacheability, MISS→HIT, never-cache rules, cloaking diff, image optimization, Core Web Vitals |
| **G. Sitemaps & errors** | generated sitemap, listing coverage, per-language alternates, true 404 vs soft-200 vs 403 |
| **H. Owner access** | what it *can't* check — Search Console, Bing Webmaster, GA4, field CWV, backlinks — reported honestly rather than guessed |

## Scope

Technical setup only. It does not assess content quality, keyword strategy, or
backlinks — and it says so rather than silently skipping them. The technical
setup makes a site *eligible* to be found; the content is what actually wins.

## Credit

The check list and its confidence tiers are derived from
[The Technical Setup Behind a Fast, AI-Discoverable Blog](https://blog.webreeze.ai/en/technical-seo-and-geo-guide).

## License

MIT
