# daily-facets.com

An OG stub. One static page, no build, no app.

Its whole job: carry a link-preview card when the bare domain gets pasted somewhere social
(Discord and the like), then bounce the visitor to today's Daily Facet.

Destination: `https://facets.bigtentgames.com/?daily`

## Why it is a page and not a redirect rule

A Cloudflare redirect rule can't carry OG tags. A page can, so this clones the pattern that
`facets/open/index.html` already proved: OG/twitter tags plus a zero-delay redirect, empty
body.

## Why the redirect is a script, not a static meta refresh

The destination depends on where the stub itself is being loaded from, so it can't be a
single hardcoded `<meta http-equiv="refresh">`: production (`daily-facets.com`) needs to land
on `https://facets.bigtentgames.com/?daily`, but a local test (`daily-facets.local`) needs to
land on the local dev app, `https://facets.bigtentgames.local/?daily` — hardcoding the prod
URL would always bounce local testing out to the live site. A tiny inline script picks the
target by `location.hostname`. **The static meta refresh survives inside a `<noscript>`,
hardcoded to production**, as the fallback for non-JS clients — OG scrapers (Discord,
Facebook, etc.) only ever read the `<head>` meta tags and don't execute the redirect either
way, so this doesn't touch the OG card.

## Why it is its own repo

Decoupled deploys. No service worker, no auth cookie, no version bump, and no waiting on the
main app's batched release train.

## Why it is not a landing page

The app's forced first-run example already teaches a newcomer in context and hands off with
"try the Daily Facet." A landing page would duplicate a purpose-built flow worse, cost conversions
as an interstitial, and add a page to maintain.

## The link is date-free on purpose

The share text names the sender's day ("July 16"); the link always resolves to the reader's
today. They drift apart by design. Don't "fix" it by dating the link, which would hand a
newcomer a stale puzzle whenever they tap late.

## Deploy

Cloudflare Workers (with static assets), connected to this repo via git. No build command.

Custom domains: add both `daily-facets.com` and `www.daily-facets.com` as Workers Custom Domains.

### Setup checklist

The domain is registered at **Network Solutions**, which shapes the order below. The apex is
the reason this isn't a two-click job: Pages can serve a custom domain whose DNS lives
elsewhere, but only via a CNAME to the `*.pages.dev` target, and **a CNAME at the apex is
illegal in DNS.** Network Solutions has no ALIAS or flattening to work around it. Since the
whole point of this domain is that people read and type the bare name, apex-broken is not
shippable. Putting DNS on Cloudflare gets CNAME flattening, which is what makes the apex work.

Registration stays at Network Solutions. Only DNS moves, and that part is free.

**Cloudflare assigns nameservers PER ZONE, not per account.** `bigtentgames.com` and
`wilvoss.com` both landing on `brodie`/`poppy` is a coincidence of the pool, not a rule.
`daily-facets.com` was added to the _same_ account and was assigned **`braden.ns.cloudflare.com`
/ `ingrid.ns.cloudflare.com`** — the SAME pair as facetsdaily.com before it, also a coincidence
of the pool. On the Free plan the pair is not selectable. So the NS pair cannot be known before
the zone is created, and delegating in advance by copying another domain's pair does not work.

1. Cloudflare → **Add a domain** (older dashboards say "Add a site") → `daily-facets.com`, Free
   plan. Do this **first** — it is what mints the NS pair. Until the zone exists, whatever
   nameservers the registry points at answer `REFUSED` (they deny knowing the domain), it
   resolves nowhere, and no amount of waiting fixes it. That failure looks exactly like
   "propagation," which is the trap.
2. Network Solutions → set the nameservers to **the pair Cloudflare just showed you**, and
   delete any others.
3. Network Solutions → **turn off domain parking / forwarding** if it's on. Usually on by
   default on a fresh registration.
4. Hit **Check nameservers now** on the activation page. **Then ignore what the page says
   next.** It quotes "1-2 hours, up to 24" both before _and_ after the check succeeds — it is
   static copy, not a live measurement.

   What actually happened here (2026-07-19): NetSol's save reached the `.com` registry in
   about a minute, the zone went **active on the click**, and 1.1.1.1 / 8.8.8.8 / 9.9.9.9 /
   OpenDNS all served the new pair immediately. Total elapsed: minutes. The page still said
   hours.

   Trust these, not the dashboard:
   - `dig +trace NS daily-facets.com` — what the `.com` servers hand out. Want: the assigned pair.
   - `dig SOA daily-facets.com @<assigned-ns>` — want an answer, **not `REFUSED`**.
   - `curl -s "https://api.cloudflare.com/client/v4/zones?name=daily-facets.com" -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"`
     — the real `status` and `activated_on`. (Token is Workers/Pages-scoped, so it can _read_
     the zone but cannot trigger `activation_check`; that returns `9109 Unauthorized`. The
     dashboard button is the only way to force it.)
5. Cloudflare dashboard → **Workers & Pages** → **Import a repository** → connect this GitHub
   repo. No build command; it deploys as a Worker with static assets, not a Pages project.
6. Worker → **Settings → Domains & Routes** → add `daily-facets.com` **and**
   `www.daily-facets.com` as Custom Domains. With the zone on Cloudflare, both records get
   created for you automatically — **unless the zone's initial scan already imported stray
   DNS records for those hostnames** (it can pull in the old registrar parking-page A records).
   If attaching a Custom Domain 400s with "already has externally managed DNS records," delete
   whatever's sitting on that hostname in DNS → Records first, then retry.
7. Verify: the bare domain redirects to the Daily Facet, and the OG card renders. Paste the link in
   Discord, or use any OG debugger. Cloudflare is authoritative now, so make DNS changes
   there, not in the NetSol panel.

### After ~mid-September, consider moving the registration to Cloudflare

ICANN locks a newly registered domain against **registrar** transfer for 60 days. Registration
was 2026-07-19 23:04 UTC, so the lock lifts around **2026-09-17**. Nameserver changes have no
such lock, which is why the split above works today.

Worth doing on the merits, not just price: Cloudflare Registrar sells at cost with no renewal
markup, and **WHOIS privacy is included free** rather than being a paid upsell. Public WHOIS
on a NetSol default means a name and address in a scrapeable database. This is the same
principle that produced the platform's own auth.

## Local testing

`daily-facets.local` resolves via the existing wildcard Apache vhost
(`ServerAlias *.local` → `VirtualDocumentRoot "/Users/wilvoss/Sites/%1"` in
`/etc/apache2/extra/httpd-vhosts.conf`) and the shared cert, which already lists every bare
`.local` hostname explicitly (the wildcard SAN itself stopped being honored by browsers) — add
`daily-facets.local` to that list if it isn't there yet. Only the `/etc/hosts` line
(`127.0.0.1 daily-facets.local`) should be needed otherwise. Visiting it locally redirects to
`https://facets.bigtentgames.local/?daily` instead of production, per the script above.

## Files

- `index.html` — the stub
- `images/facets_og.png` — 1200x630 preview card, **copied** from the app (`facets`
  → `public/images/facets_og.png`), not hotlinked, so this repo stays self-contained and a
  Daily Facet-specific card can replace it without an app deploy. **The cost: the two copies drift
  independently. Re-copy on every re-render of the app's card,** or this serves a stale one.
  Compare md5s, not timestamps.
- `favicon.svg`
