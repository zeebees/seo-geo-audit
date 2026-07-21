# Check list

Every check: the command to run, what a pass looks like, and a confidence tier.
Run all of them. Report all of them, passes included.

Set once:

```bash
HOST=example.com          # canonical host, no scheme
ART=https://example.com/en/some-post     # a representative article
IDX=https://example.com/en               # the listing/index page
```

## Facts that changed recently — trust these over training data

- **Google retired FAQ rich results from Search entirely in May 2026.** The 2023
  change had already limited them to government and health sites; they are now
  gone for everyone. `FAQPage` markup is still valid Schema.org and Google still
  reads it — it just earns no expandable Q&A. Never report missing FAQ schema as
  a lost rich result.
- **GA4 added an `AI Assistant` channel on 13 May 2026** — AI referral traffic is
  attributed automatically, no tagging. It is a *floor, not a total*: referrer-less
  AI clicks land in Direct, and Google's AI Overviews / AI Mode count as Organic.
- **`Google-Extended` is not a crawler.** It is a control token governing Google's
  model training and some Gemini grounding. It fetches nothing and has zero effect
  on Google Search. Do not report it as a blocked crawler.
- **Google's Indexing API is limited to job postings and livestreams.** There is no
  per-URL push for ordinary pages. IndexNow does *not* reach Google.

---

## A. Hostname & canonical

**A1. Both `www` and bare apex resolve** `[proven]`
```bash
curl -sSI -o /dev/null -w '%{http_code} %{redirect_url}\n' https://$HOST/
curl -sSI -o /dev/null -w '%{http_code} %{redirect_url}\n' https://www.$HOST/
```
Pass: both respond, one 301s to the other. **Fail if either does not resolve at
all** — a missing apex DNS record is the common one, and it dead-ends every
typed visit and every backlink to that form. This is the single most-missed check.

**A2. Only one host serves 200** `[proven]`
Both forms returning `200` splits credit. Pick one, 301 the other.

**A3. Self-referential canonical on every page** `[proven]`
```bash
curl -sS $ART | grep -i 'rel="canonical"'
```
Pass: absolute URL, matches the fetched URL, present on article *and* index.

**A4. Trailing-slash and query variants fold to canonical** `[proven]`
```bash
curl -sSI -o /dev/null -w '%{http_code}\n' "$ART/"
curl -sS "$ART?utm_source=x" | grep -i 'rel="canonical"'
```
Pass: slash variant redirects or canonicalizes; `?utm=` variant stays 200 with a
clean canonical (that is what folds it).

**A5. Site root redirect is permanent** `[contested]`
A `307`/`302` from `/` to `/en/` does not consolidate signals. Prefer `301`/`308`
— *unless* the default language genuinely changes, where temporary is correct.
Ask before calling it a fail.

**A6. Plain HTTP redirects to HTTPS** `[proven]`
```bash
curl -sSI --max-time 15 -o /dev/null -w '%{http_code} %{redirect_url}\n' http://$HOST/
```
Pass: `301` to `https://`. A hang or empty response is a P1 — link checkers and
crawlers that probe HTTP first get nothing.

**A7. HSTS header present** `[proven]`
```bash
curl -sSI https://$HOST/ | grep -i strict-transport-security
```
Pass: `max-age` of at least six months. Absence is `P3` — the HTTP→HTTPS redirect
in A6 is what actually matters; HSTS removes the first-request round trip.

## B. Languages

**B1. Every language has its own URL** `[proven]`
The highest-value check in this file. A site can *look* bilingual and be
monolingual to every crawler.
```bash
curl -sS $ART | grep -oE 'data-lang="[a-z-]+"' | sort -u
curl -sS $ART | grep -oE '<html[^>]*lang="[^"]*"'
```
Fail if translations live at the same URL behind a JS toggle (`data-lang` blocks,
a hardcoded `<html lang>`, `og:locale:alternate` with no distinct URLs). The
non-default language is then unrankable and uncitable, and the bilingual signals
advertise a capability no crawler can reach. A language *menu* is fine — it just
has to navigate to a different URL.

**B2. `hreflang` is self-referential and reciprocal** `[proven]`
```bash
curl -sS $ART | grep -i 'rel="alternate"'
```
Pass: every version lists every version *including itself*, valid codes, and the
same set appears on each translation.

**B3. `x-default` present where there is a real fallback** `[proven]`
Recommended, not required. Should point at the fallback for untargeted languages.

**B4. Only existing translations are advertised** `[proven]`
Every `hreflang` target must return 200. Pointing a crawler at a translation that
does not exist is worse than omitting it.

## C. Page basics

**C1. Exactly one `<h1>`** `[contested]`
```bash
curl -sS $ART | grep -oc '<h1'
```
Google says heading order is not a ranking factor, but two `<h1>`s is real
ambiguity. Classic cause: the CMS renders the title *and* the body starts with a
Markdown `# Title`. Fix by demoting body headings at render time.

**C2. Unique `<title>` and meta description per page** `[proven]`
```bash
for u in "$ART" "$IDX"; do curl -sS "$u" | grep -iE '<title>|name="description"'; done
```
Fail on a site-wide default, a missing description, or literal newlines inside
the attribute. Note these *influence* but do not control the SERP — Google
frequently rewrites both.

**C3. Semantic landmarks** `[proven]`
```bash
curl -sS $ART | grep -oE '<(article|main|nav|header|footer)[ >]' | sort | uniq -c
```
Proven win is accessibility; crawler parsing is the secondary benefit.

**C4. Headings carry `id` anchors** `[bet]`
Deep anchors let answer engines and Google passage features cite a *section*.
Auto-slug H2–H4. Cheap; unmeasured.

**C5. Content is server-rendered** `[proven]`
```bash
curl -sS -A 'Mozilla/5.0 (compatible; Googlebot/2.1)' $ART \
  | tee /tmp/raw.html | wc -c
grep -oE '<div id="(root|app|__next)"></div>' /tmp/raw.html
grep -c '<h1' /tmp/raw.html
```
The highest-severity check in this section. A client-rendered SPA returns a few
hundred bytes and an empty mount node — no `<h1>`, no body text. Report the byte
count and the empty div verbatim; it is the most legible evidence in the report.
Google may eventually render it; **answer-engine crawlers largely do not**, so a
CSR site is close to invisible to citation regardless of content quality. Fix is
SSR, SSG, or prerendering for bots.

**C6. Open Graph and Twitter card completeness** `[proven]`
```bash
curl -sS $ART | grep -oE '<meta (property="og:[^"]+"|name="twitter:[^"]+")[^>]*>'
```
Want `og:title`, `og:description`, `og:type=article`, `og:url`, `og:image`,
`article:published_time`/`modified_time`. Check the image is under ~300 KB —
several scrapers degrade or skip above that.

**C7. `robots` meta permits full snippets** `[proven]`
```bash
curl -sS $ART | grep -oE '<meta name="robots"[^>]*>'
```
Absent is fine (defaults are permissive). Fail on `nosnippet`, `noarchive`, or a
low `max-snippet`, which cap what search *and* AI surfaces may quote from you.
`max-snippet:-1, max-image-preview:large` is the maximally-quotable setting.

**C8. Images have meaningful `alt`** `[proven]`
```bash
curl -sS $ART | grep -oE '<img[^>]*>' | grep -cv 'alt="[^"]\+"'
```
Proven accessibility win; also the only text description a crawler gets of an image.

**C9. Mobile viewport declared** `[proven]`
```bash
curl -sS $ART | grep -oE '<meta name="viewport"[^>]*>'
```
Pass: `width=device-width, initial-scale=1`. Absence is a mobile-usability failure.

## D. Structured data

**D1. Valid JSON-LD on every page** `[proven]`
```bash
curl -sS $ART | sed -n 's/.*<script type="application\/ld+json">\(.*\)/\1/p' | head -5
```
Pass: parses as JSON. Article pages want `BlogPosting` (headline, datePublished,
dateModified, author, publisher, `mainEntityOfPage`, `inLanguage`, image) plus
`BreadcrumbList`; index wants `WebSite` + `ItemList`.

**D2. `</` escaped as `<\/` inside JSON-LD** `[proven]`
A literal `</script>` in content breaks out of the block and silently kills it.

**D3. Named author entity for YMYL topics** `[contested]`
Health, finance, legal. Google frames this as E-E-A-T and recommends accurate
bylines where readers expect them. A `Person` author with `url` and credentials
beats `Organization`. Not a single ranking dial — do not overclaim.

**D4. `FAQPage` where genuine Q&A exists** `[bet]`
Optional. **Earns nothing in Google Search since May 2026** (see facts above).
Remaining value is AI extractability — a reasonable bet, not a measured win.
Never force it onto content that is not really Q&A.

**D5. `sameAs` on the `Organization`** `[contested]`
Low priority. `sameAs` is a *claim, not proof* — Google documents it as links to
more information and does not require backlinks. Field wisdom: only profiles that
**link back** can be corroborated. Skip entirely if the business has little online
presence; it connects identities, it cannot create them.

## E. AI crawler access

**E1. `robots.txt` allows the answer crawlers** `[proven]`
```bash
curl -sS https://$HOST/robots.txt
```
The three that matter for citation: `OAI-SearchBot`, `PerplexityBot`,
`Claude-SearchBot`. Distinguish three classes and do not conflate them:
- *Answer crawlers* — the ones you want.
- *User-triggered fetchers* — `ChatGPT-User`, `Claude-User`. Vendors say robots
  rules may not fully apply, since a human initiated the fetch.
- *Training crawlers* — `GPTBot`, `ClaudeBot`, `CCBot`. Allowing these is a
  separate, values-based decision that does **not** affect whether you get cited
  today. Blocking them limits *future* training only; it cannot unwind snapshots
  already taken.

**E2. Answer crawlers actually get through** `[proven]`
The check the `robots.txt` read cannot answer. Many CDN/WAF products ship an
AI-bot block that `robots.txt` cannot override.
```bash
for UA in OAI-SearchBot PerplexityBot Claude-SearchBot; do
  printf '%s ' "$UA"
  curl -sS -A "$UA" -o /dev/null -w '%{http_code}\n' $ART
done
```
Fail on any `403`/`429`/challenge. A permissive `robots.txt` with a blocking
firewall is a silent, total citation failure.

**E3. No `noindex` on the AI-facing copies** `[proven]`
```bash
curl -sSI $ART.md | grep -i 'x-robots-tag'
curl -sS $ART | grep -i 'name="robots"'
```
**OpenAI and Anthropic each honor `noindex` as a signal to keep content out of
their AI search too.** So a `noindex` added to a clean `.md` alternate to dodge
duplicate content hides it from the exact engines it was built to feed. Handle
duplication with a canonical instead — for a plain `.md` that means an HTTP
`Link: rel="canonical"` header, not an HTML tag — or drop the alternate entirely.

**E4. `.md` alternate serves real Markdown** `[bet]`
```bash
curl -sS $ART.md | head -20
```
If it advertises clean Markdown and emits HTML, the claim is false and it burns
tokens on markup for no gain.

**E5. `llms.txt` present** `[bet]`
```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://$HOST/llms.txt
```
An emerging convention, near-zero cost. **No public evidence any major answer
engine reads it** — they document ordinary crawling and `robots.txt`. Report as a
cheap bet, never as a lever that pays off today. If present, check it is written
in the site's primary language.

**E6. IndexNow key file** `[proven]`
```bash
curl -sS https://$HOST/robots.txt | grep -i indexnow
```
Notifies Bing, Yandex and a few others; they still decide whether to crawl.
**Google is not a participant** — never imply IndexNow helps Google.

**E7. RSS or Atom feed** `[bet]`
```bash
for f in /feed.xml /rss.xml /atom.xml /feed; do
  printf '%s ' "$f"; curl -sS -o /dev/null -w '%{http_code}\n' "https://$HOST$f"
done
```
Feeds remain a live discovery channel for aggregators and several AI ingestion
pipelines. `ABSENT`, never `FAIL`.

## F. Speed & caching

**F1. Public HTML is edge-cacheable** `[proven]`
```bash
curl -sSI $ART | grep -iE 'cache-control|cf-cache-status|x-cache|age'
```
Pass: `s-maxage` (or CDN rule) on the shared copy, with a revalidating browser
copy so a purge is globally effective. `no-store` on public article HTML is a
straight loss — a common cause is an auth-app default leaking onto the blog host.

**F2. Second fetch is a HIT** `[proven]`
```bash
curl -sSI $ART | grep -i 'cf-cache-status\|x-cache'
curl -sSI $ART | grep -i 'cf-cache-status\|x-cache'
```
Pass: MISS then HIT at a warm edge.

**F3. Errors and personalized pages are not cached** `[proven]`
```bash
curl -sSI https://$HOST/definitely-not-a-real-page-xyz | grep -iE 'cache|http/'
```
404s want BYPASS or a very short TTL, so a transient error is not pinned in place.
Anything varying by session/login/token must stay dynamic. Cache on
**public-and-identical**, not on status code — a personalized page or soft-404
can also return 200.

**F4. Bot and human get byte-identical HTML** `[proven]`
```bash
diff <(curl -sS -A 'Mozilla/5.0' $ART) <(curl -sS -A 'OAI-SearchBot' $ART) | head
```
Any difference is cloaking. Empty diff is the pass.

**F5. Images optimized and on a CDN** `[proven]`
```bash
curl -sSI "$(curl -sS $ART | grep -oE 'rel="preload"[^>]*as="image"[^>]*href="[^"]+"' | grep -oE 'https?://[^"]+')" \
  | grep -iE 'content-length|content-type|cache-control'
```
Heavy generated images (heroes, illustrations) belong on a CDN with `srcset` and
AVIF/WebP. Content-addressed filenames want `max-age=31536000, immutable`.
Already-fast framework-served static assets do **not** need moving — that adds
complexity for no gain.

**F6. Core Web Vitals thresholds** `[proven]`
LCP < 2.5s · INP < 200ms · CLS < 0.1. Lab numbers only from a fetch; real-user
data needs owner access — put that in the could-not-check list.

## G. Sitemaps & error pages

**G1. `sitemap.xml` exists, is generated, and is referenced in `robots.txt`** `[proven]`
```bash
curl -sS https://$HOST/sitemap.xml | head -30
```
Fail on hand-maintained drift. `<lastmod>` should reflect a real content change,
not every render.

**G2. Sitemap covers every page including listings** `[proven]`
Listing/index pages are commonly omitted despite carrying `ItemList` schema.

**G3. Per-language alternates in the sitemap** `[proven]`
Declare the XHTML namespace on the root, and give *every* language its own
`<url>` entry repeating the full alternate set — not one entry for all.

**G4. Missing pages return a true 404** `[proven]`
```bash
curl -sSI -o /dev/null -w '%{http_code}\n' https://$HOST/definitely-not-a-real-page-xyz
```
Pass: `404` or `410`. Fail on a soft-`200` (lingers as a phantom URL) or a `403`
(claims forbidden, which is not what is true). The reason is *correct description*,
not crawl budget — Google says missing-page responses do not hurt crawl rate.
Common causes: a static host 403ing missing files, or an SPA 200ing everything.

**G5. `<lastmod>` reflects real content change** `[proven]`
```bash
curl -sS https://$HOST/sitemap.xml | grep -oE '<lastmod>[^<]+' | sort -u | head
```
Every URL sharing today's date means it is stamped at render, not at edit — the
signal is then noise and crawlers learn to ignore it.

**G6. Articles carry contextual internal links** `[proven]`
```bash
curl -sS $ART | grep -oE 'href="[^"]+"' | grep -c "$HOST\|href=\"/"
```
Distinguish chrome (header, breadcrumb, language switcher) from in-body
contextual links. Zero contextual internal links orphans the page — it inherits
no authority and gives a crawler no path onward. On a one-article site this is
structural rather than negligent; say so.

## H. Requires owner access — do not guess

List these as could-not-check, with what they would reveal:

- Google Search Console — verification, sitemap submission, coverage, queries
- Bing Webmaster Tools — manual setup; pairs with IndexNow for the Bing/AI side
- GA4 `AI Assistant` channel — under Acquisition → Traffic acquisition
- Real-user Core Web Vitals (field data, not lab)
- Backlinks and off-site authority — the slowest and often highest lever
- Live AI-citation probes — asking the engines the questions the content should own
