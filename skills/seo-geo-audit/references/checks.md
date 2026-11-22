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
Pass: both respond, one 301s to the other. **Fail if exactly one does not resolve**
— a missing apex DNS record is the common one, and it dead-ends every typed visit
and every backlink to that form. This is the single most-missed check.

**`N/A` on a subdomain.** Only a registrable domain has a `www` twin. Auditing
`blog.example.com` would probe `www.blog.example.com`, which no sane DNS defines —
that is not a missing record. Watch two-label suffixes: `example.com.au` and
`example.co.uk` *are* apexes, so A1 still applies to them. If **neither** form
responds, report `N/A` too: you cannot tell a dead site from your own blocked or
offline fetch. The working form is what proves the other is genuinely dead.

**A2. Only one host serves 200** `[proven]`
Both forms returning `200` splits credit. Pick one, 301 the other.

**A3. Self-referential canonical on every page** `[proven]`
```bash
curl -sS $ART | grep -i 'rel="canonical"'
```
Pass: a canonical is present on article *and* index and is self-referential (same
page as the fetched URL). Absolute is preferred, but a **relative** canonical
(`href="/path"`) is spec-valid and resolved by Google — accept it. Fail only if the
tag is absent or points at a *different* page.

**A4. Trailing-slash and query variants fold to canonical** `[proven]`
```bash
curl -sSI -o /dev/null -w '%{http_code}\n' "$ART/"
curl -sS "$ART?utm_source=x" | grep -i 'rel="canonical"'
```
Pass: slash variant redirects or canonicalizes; `?utm=` variant stays 200 with a
clean canonical (that is what folds it). If you did not probe the variants and the
page simply has no canonical, that is a `P3` `WARN`, not a `FAIL` — server-side 301s
are an equally valid way to fold variants.

**A5. Site root redirect is permanent** `[contested]`
A `307`/`302` from `/` to `/en/` does not consolidate signals. Prefer `301`/`308`
— *unless* the default language genuinely changes, where temporary is correct.
Ask before calling it a fail.

**A6. Plain HTTP redirects to HTTPS** `[proven]`
```bash
curl -sSI --max-time 15 -o /dev/null -w '%{http_code} %{redirect_url}\n' http://$HOST/
```
Pass: `301` to `https://`. Fail `P1` only when plain HTTP **serves a response**
without upgrading — link checkers and crawlers that probe HTTP first never reach
the canonical page. No response at all (port 80 closed, connection refused, a
timeout) is `N/A`: HTTPS-only is often deliberate, and a transient failure proves
nothing. Say it was inconclusive.

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
curl -sS $ART | grep -oiE '<link[^>]*rel="alternate"[^>]*hreflang="[^"]+"' | sort -u
curl -sS $ART | grep -oE '<meta[^>]*property="og:locale:alternate"[^>]*>'
```
Fail if translations are advertised (`og:locale:alternate`, or `hreflang` that all
resolves to one URL) but live at the same URL behind a JS toggle. **Do not use a
raw `data-lang` count** — code-syntax highlighters emit `data-lang="js"` (and
wrapper `<div data-lang><pre>`), which can't be told apart from a translation block
and false-flags code blogs. The non-default language is then unrankable and
uncitable, and the bilingual signals
advertise a capability no crawler can reach. A language *menu* is fine — it just
has to navigate to a different URL.

**B2. `hreflang` is self-referential and reciprocal** `[proven]`
```bash
curl -sS $ART | grep -i 'rel="alternate"'
```
Pass: every version lists every version *including itself*, valid codes, and the
same set appears on each translation. A regional page language counts as covered by
its base-language entry: a page with `<html lang="en-US">` that lists itself as
`hreflang="en"` is self-referential. Fail only when a language has no entry at all.

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
curl -sS -A 'Mozilla/5.0 (compatible; Googlebot/2.1)' $ART > /tmp/raw.html
wc -c < /tmp/raw.html
grep -oE '<div id="(root|app|__next)"></div>' /tmp/raw.html
grep -c '<h1' /tmp/raw.html
# visible text inside <body> only — the number that decides this check
perl -0777 -ne 's/.*?<body[^>]*>//is; s/<!--.*?-->//gs;
  s/<(script|style|noscript|template)\b.*?<\/\1>//gis; s/<[^>]+>/ /g;
  s/\s+/ /g; print length' /tmp/raw.html
```
The highest-severity check in this section. A client-rendered SPA returns an empty
mount node — no `<h1>`, no body text. **Judge on visible `<body>` text, never on
page bytes.** Analytics, pixel and affiliate tags in `<head>` routinely push an
empty shell past 6 KB, so a byte threshold false-passes exactly the sites this
check exists to catch. Fail when the body carries under ~200 characters of visible
text *and* there is an empty mount node or no `<h1>`. Report the body-text count
and the empty div verbatim; it is the most legible evidence in the report.

Before failing C5, confirm `$ART` is an HTML page: check the response
`content-type` and that the body has a `<body>` tag. A few hundred bytes with no
`<body>` is a sitemap or text file picked by mistake (see the sitemap-index note
under G1), not an empty shell.

When C5 fails, every other check that reads the page (canonical, headings,
JSON-LD, Open Graph, internal links) is judging the empty shell. Scope those rows
to **the delivered HTML** — "the delivered HTML has no canonical tag", never "the
site has no canonical tag" — and note JavaScript may add them later. Serving
rendered HTML is the one fix that delivers all of them; do not present each as a
separate missing feature.
Google may eventually render it; **answer-engine crawlers largely do not**, so a
CSR site is close to invisible to citation regardless of content quality. Fix is
SSR, SSG, or prerendering for bots.

**C6. Open Graph and Twitter card completeness** `[proven]`
```bash
curl -sS $ART | grep -oE '<meta (property="og:[^"]+"|name="twitter:[^"]+")[^>]*>'
```
Want `og:title`, `og:description`, `og:type=article`, `og:url`, `og:image`,
`article:published_time`/`modified_time`. Check the image is under ~300 KB —
several scrapers degrade or skip above that. Count `og:*` tags written with
`name=` instead of `property=` (`<meta name="og:url" …>`) as present; the major
scrapers accept both, so grep for either attribute before calling a tag missing.

**C7. `robots` meta permits full snippets** `[proven]`
```bash
curl -sS $ART | grep -oE '<meta name="robots"[^>]*>'
```
Absent is fine (defaults are permissive). Fail on `nosnippet` or `max-snippet:0`,
which cap what search *and* AI surfaces may quote from you. **Not `noarchive`** — it
only suppresses the cached-page link; snippets still show, so it is not a snippet
cap. `max-snippet:-1, max-image-preview:large` is the maximally-quotable setting.

**C8. Images have meaningful `alt`** `[proven]`
```bash
curl -sS $ART | grep -oE '<img[^>]*>' | grep -cv 'alt='
```
Count only images with **no `alt` attribute at all**. An explicit `alt=""` is the
valid, intentional marker for a *decorative* image — do not flag it. Also skip
**tracking beacons**: an `<img>` inside `<noscript>` (Meta Pixel ships one), a
`width="1" height="1"` pixel, or one styled `display:none`. They are not content
images, and flagging them is a false finding on every site running ads. Proven
accessibility win; also the only text description a crawler gets of an image.

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
Pass: parses as JSON. Recommend the schema that fits **the page type** — do not
assume every page is a blog post. Article pages want `BlogPosting` (headline,
datePublished, dateModified, author, publisher, `mainEntityOfPage`, `inLanguage`,
image) plus `BreadcrumbList`; a blog index wants `WebSite` + `ItemList`; the site
root (or a bare locale root like `/en`) wants `WebSite` + `Organization`; a
product or service page wants `Product`/`Service`. Do not suggest a sitelinks
`SearchAction` — Google retired the sitelinks search box in November 2024.

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
Read `Allow:` lines too, with Google's precedence: the **longest matching rule
wins**, and a bot with no group of its own follows `User-agent: *`. An allowlist
file — `Disallow: /` plus `Allow: /$` and `Allow: /blog/` — does *not* block the
root or the blog. Ignoring `Allow:` turns it into a false P1.

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
A `403`/`429`/challenge here is a **WARN, not a hard fail**. The answer engines
verify their crawlers by source IP / reverse-DNS, so a site can correctly allow the
real bots while blocking a spoofed UA sent from an arbitrary IP (like this one). It
cannot confirm the *real* crawler is blocked — say so, and tell the owner to check
their server logs (real crawler IPs) or the engine's own fetch tool. (Pass only on
`2xx`; a `404`/`5xx`/timeout is also inconclusive, not a pass.)

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
tokens on markup for no gain. Two cases are not failures:
- **The page advertises no Markdown copy.** Only a page that links one
  (`<link rel="alternate" type="text/markdown" …>`) makes a claim. If `$ART.md`
  redirects to the HTML page, or returns the same HTML because the app ignores the
  suffix (try `$ART.xyz`: a 200 there too means a catch-all), there is no Markdown
  alternate at all: report it absent (`P3`), not a false claim.
- **The response is `content-type: text/markdown`** but contains `<html>` or
  `<div>` inside a code sample. That is real Markdown documenting HTML; pass it.

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
  printf '%s ' "$f"; curl -sS "https://$HOST$f" | head -c 300 | tr '\n' ' '; echo
done
```
A `200` is not a feed. An SPA answers every path, `/feed` included, with its HTML
shell. Count a feed only if the body's root element is `<rss`, `<feed` or
`<rdf:RDF` (after any `<?xml …?>` prolog). Feeds remain a live discovery channel
for aggregators and several AI ingestion pipelines. `ABSENT`, never `FAIL`.

## F. Speed & caching

**F1. Public HTML is edge-cacheable** `[proven]`
```bash
curl -sSI $ART | grep -iE 'cache-control|cf-cache-status|x-cache|age'
```
Pass: `s-maxage` (or CDN rule) on the shared copy, with a revalidating browser
copy so a purge is globally effective. `no-store` on public HTML is a straight
loss — a common cause is an auth-app default leaking onto a public host.

**Check the edge before blaming the origin header.** A CDN rule can override the
origin's `Cache-Control` entirely (Cloudflare's "override origin" exists for this).
If either fetch shows `cf-cache-status: HIT` or `x-cache: HIT`, the page *is* being
cached, whatever the origin says. That is `PASS*`: it works, but the origin header
no longer describes the behaviour, so keep the two in sync if the rule is removed.

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
can also return 200. If the missing-page probe **redirected** (you are now looking
at the destination's headers) or got no response, this is `N/A`, not a verdict.

**F4. Bot and human get the same content** `[proven]`
```bash
diff <(curl -sS -A 'Mozilla/5.0' $ART) <(curl -sS -A 'OAI-SearchBot' $ART) | head
```
Compare the *content*, not the bytes. A rotating CSP nonce, CSRF token, or
timestamp makes two correct responses differ — that is **not** cloaking. Fail only
on a material divergence: a different `<title>`/`<h1>`, or the bot getting a
stripped/stub body (a challenge page, or far less text). Ignore per-request noise.
If the bot fetch itself failed (`403`/`429`/timeout, or a `204`/`205` with no
body), there is nothing to compare: `N/A`, and E2 reports the block. A failed
fetch is not cloaking.
If the bot response is `content-type: text/markdown` (some sites serve AI crawlers
a Markdown copy of the page), that is **not** cloaking; it is the token-efficient
ideal. Compare the Markdown's length to the human page's visible text and pass it
(with a note) unless it collapses to a stub, under about half the human text.

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
curl -sS https://$HOST/robots.txt | grep -i '^sitemap:'   # the AUTHORITATIVE URL
curl -sS https://$HOST/sitemap.xml | head -30             # the conventional path
```
Honor the URL **declared in `robots.txt`** first — the path is case-sensitive (many
sites serve `/Sitemap.xml`, not `/sitemap.xml`) and may sit on a different origin
such as a CDN — the owner declared it, so a cross-origin sitemap counts. Only
call it missing if *neither* the declared URL nor `/sitemap.xml` returns a sitemap.
Fail on hand-maintained drift. `<lastmod>` should reflect a real content change,
not every render.

**Sitemap index.** If the root element is `<sitemapindex>`, the `<loc>` entries are
child sitemaps, not pages — the shape every Shopify store and most Yoast WordPress
sites serve. Fetch the children (the first five are enough for sampling) and read
page URLs from them:
```bash
curl -sS https://$HOST/sitemap.xml | grep -c '<sitemapindex'      # 1 = an index
curl -sS https://$HOST/sitemap.xml | grep -oE '<loc>[^<]+' | sed 's/<loc>//; s/&amp;/\&/g' | head -5 |
  while read -r child; do curl -sS "$child" | grep -oE '<loc>[^<]+' | sed 's/<loc>//'; done | head -40
```
`<loc>` is XML, so un-escape `&amp;` before fetching (Shopify child URLs carry
`?from=…&amp;to=…`). Choose `$ART` and the listing from the children's URLs, and
skip entries that are not HTML pages: `.xml`, `.md` (Shopify lists `/agents.md`),
`.txt`, `.json`, `.pdf`. Treating a child sitemap as the content page is how a
fully server-rendered store gets a false "empty shell" on C5, plus phantom
missing-title, canonical, Open Graph, JSON-LD and viewport failures.

**G2. Sitemap covers every page including listings** `[proven]`
Listing/index pages are commonly omitted despite carrying `ItemList` schema.
On a sitemap index, count the `<url>` entries in the **children**; the index's own
`<loc>` count is the number of child sitemaps, not pages.

**G3. Per-language alternates in the sitemap** `[proven]`
Declare the XHTML namespace on the root, and give *every* language its own
`<url>` entry repeating the full alternate set — not one entry for all.
On a sitemap index, look for `xhtml:link` in the children. **Do not warn when the
pages already declare `hreflang` alternates in their HTML** — that is a complete
declaration on its own (Shopify ships per-locale sitemaps plus on-page hreflang and
never `xhtml:link`). Report it as a pass with a note that sitemap alternates are
optional redundancy.

**G4. Missing pages return a true 404** `[proven]`
```bash
curl -sSI -o /dev/null -w '%{http_code}\n' https://$HOST/definitely-not-a-real-page-xyz
```
Pass: `404` or `410`. Fail on a soft-`200` (lingers as a phantom URL) or a `403`
(claims forbidden, which is not what is true). The reason is *correct description*,
not crawl budget — Google says missing-page responses do not hurt crawl rate.
Common causes: a static host 403ing missing files, or an SPA 200ing everything.
Use `-w '%{http_code}'` **without** `-L`. If the probe redirects (to the homepage,
say), the redirect is the answer to report, not a soft-404: following it reads the
destination's `200`. Redirected or failed probes are `N/A`.

**G5. `<lastmod>` reflects real content change** `[proven]`
```bash
curl -sS https://$HOST/sitemap.xml | grep -oE '<lastmod>[^<]+' | sort -u | head
```
Every URL sharing today's date means it is stamped at render, not at edit — the
signal is then noise and crawlers learn to ignore it.
On a sitemap index, read `<lastmod>` from the **children**. Shopify's index carries
none while every child entry has one, so "no `<lastmod>`" judged on the index alone
is wrong.

**G6. Pages carry contextual internal links** `[proven]`
```bash
curl -sS $ART | grep -oE 'href="[^"]+"' \
  | grep -vE 'href="(#|mailto:|tel:|//)' \
  | grep -cE "href=\"/|href=\"https?://[^/]*$HOST/"
```
Count links to the **same host** only — root-relative (`/…`) or absolute on
`$HOST`. Exclude `#`, `mailto:`, `tel:`, protocol-relative `//other`, and off-host
absolutes; a bare `//external` or a host that only appears as a substring is *not*
internal. Distinguish chrome (header, breadcrumb, language switcher) from in-body
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
