# facetsdaily.com

An OG stub. One static page, no build, no app.

Its whole job: carry a link-preview card when the bare domain gets pasted somewhere social
(Discord and the like), then bounce the visitor to today's Facets Daily.

Destination: `https://facets.bigtentgames.com/?daily`

## Why it is a page and not a redirect rule

A Cloudflare redirect rule can't carry OG tags. A page can, so this clones the pattern that
`facets/open/index.html` already proved: OG/twitter tags plus a zero-delay
`<meta http-equiv="refresh">`, empty body.

## Why it is its own repo

Decoupled deploys. No service worker, no auth cookie, no version bump, and no waiting on the
main app's batched release train.

## Why it is not a landing page

The app's forced first-run example already teaches a newcomer in context and hands off with
"try the daily." A landing page would duplicate a purpose-built flow worse, cost conversions
as an interstitial, and add a page to maintain.

## The link is date-free on purpose

The share text names the sender's day ("July 16"); the link always resolves to the reader's
today. They drift apart by design. Don't "fix" it by dating the link, which would hand a
newcomer a stale puzzle whenever they tap late.

## Deploy

Cloudflare Pages, connected to this repo. No build command, output directory `/`.

Custom domains: add both `facetsdaily.com` and `www.facetsdaily.com`.

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
`facetsdaily.com` was added to the *same* account and was assigned **`braden.ns.cloudflare.com`
/ `ingrid.ns.cloudflare.com`**. On the Free plan the pair is not selectable. So the NS pair
cannot be known before the zone is created, and delegating in advance by copying another
domain's pair does not work.

1. Cloudflare → **Add a domain** (older dashboards say "Add a site") → `facetsdaily.com`, Free
   plan. Do this **first** — it is what mints the NS pair. Until the zone exists, whatever
   nameservers the registry points at answer `REFUSED` (they deny knowing the domain), it
   resolves nowhere, and no amount of waiting fixes it. That failure looks exactly like
   "propagation," which is the trap.
2. Network Solutions → set the nameservers to **the pair Cloudflare just showed you**, and
   delete any others.
3. Network Solutions → **turn off domain parking / forwarding** if it's on. Usually on by
   default on a fresh registration.
4. Hit **Check nameservers now** on the activation page. **Then ignore what the page says
   next.** It quotes "1-2 hours, up to 24" both before *and* after the check succeeds — it is
   static copy, not a live measurement.

   What actually happened here (2026-07-16): NetSol's save reached the `.com` registry in
   about a minute, the zone went **active on the click**, and 1.1.1.1 / 8.8.8.8 / 9.9.9.9 /
   OpenDNS all served the new pair immediately. Total elapsed: minutes. The page still said
   hours.

   Trust these, not the dashboard:
   - `dig +trace NS facetsdaily.com` — what the `.com` servers hand out. Want: the assigned pair.
   - `dig SOA facetsdaily.com @<assigned-ns>` — want an answer, **not `REFUSED`**.
   - `curl -s "https://api.cloudflare.com/client/v4/zones?name=facetsdaily.com" -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"`
     — the real `status` and `activated_on`. (Token is Workers/Pages-scoped, so it can *read*
     the zone but cannot trigger `activation_check`; that returns `9109 Unauthorized`. The
     dashboard button is the only way to force it.)
4. Cloudflare Pages → create the project from the GitHub repo. No build command, output
   directory `/`.
5. Pages → **Custom domains** → add `facetsdaily.com` **and** `www.facetsdaily.com`. With the
   zone on Cloudflare, both records get created for you.
6. Verify: the bare domain redirects to the daily, and the OG card renders. Paste the link in
   Discord, or use any OG debugger. Cloudflare is authoritative now, so make DNS changes
   there, not in the NetSol panel.

### After ~mid-September, consider moving the registration to Cloudflare

ICANN locks a newly registered domain against **registrar** transfer for 60 days. Registration
was 2026-07-17 00:09 UTC, so the lock lifts around **2026-09-15**. Nameserver changes have no
such lock, which is why the split above works today.

Worth doing on the merits, not just price: Cloudflare Registrar sells at cost with no renewal
markup, and **WHOIS privacy is included free** rather than being a paid upsell. Public WHOIS
on a NetSol default means a name and address in a scrapeable database. This is the same
principle that produced the platform's own auth.

## Files

- `index.html` — the stub
- `images/facets_og.png` — 1200x630 preview card, **copied** from the app (`facets`
  → `public/images/facets_og.png`), not hotlinked, so this repo stays self-contained and a
  Daily-specific card can replace it without an app deploy. **The cost: the two copies drift
  independently. Re-copy on every re-render of the app's card,** or this serves a stale one.
  Compare md5s, not timestamps.
- `favicon.svg`
