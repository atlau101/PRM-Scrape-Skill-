# PRM Fingerprints

How to tell what software runs a partner portal, from the raw HTTP response.

**This file is meant to grow.** Vendors rebrand, CDNs change, CSP headers get rewritten. Zift
became Unifyr and both names still appear in the same page source mid-migration. When a portal
matches nothing, record the unmatched signals and add a row here.

Two tiers below. **Verified** rows were confirmed against live portals — the companies are listed
so you can re-check. **Unverified** rows are pattern guesses from vendor infrastructure and have
*not* been seen on a real customer portal. Never report an Unverified match as `Confirmed`.

Confirmed on 2026-09-08, then extended on 2026-09-09 by a 332-company sweep of Suger's `Customer`
accounts. That run promoted four Unverified rows to Verified, corrected the Magentrix pattern
(which had never matched anything), and added seven vendors the table did not know about.

---

## Where to look

In priority order — the strongest signals are the ones nobody bothers to hide:

0. **DNS `CNAME` record** — where the hostname actually points. **Check this first.** It is one
   `dig` call, costs no HTTP request, and keeps working when a WAF blocks the page body or a CDN
   masks the headers. Amadeus is fronted by Imperva and returns an empty body to automated
   requests; its CNAME still names Salesforce outright.
1. **`content-security-policy` header** — lists every domain the page may load from. A PRM vendor's
   CDN is almost always in it. Strongest single signal, and it comes back from a `HEAD` request.
2. **`server` / `x-powered-by` headers** — sometimes names the platform outright.
3. **`set-cookie` names** — platform-specific cookie names leak the stack.
4. **HTML root attributes and meta tags** — `<html ng-app="...">`, `<meta name="author">`.
5. **Script `src` hosts** — the CDN serving the app bundle.
6. **URL path shape after redirect** — e.g. `/s/` for Salesforce. Weakest, but survives when a CDN
   masks the headers.

```bash
dig +short "$PORTAL_HOST" CNAME
```

---

## DNS CNAME quick table

The fastest path to a verdict. One lookup per host.

| CNAME contains | Vendor | Confirmed at |
|---|---|---|
| `.partner-experience.com` **or** `impartner.live` | **Impartner** | GitLab, Vanta, Axonius, Syncari, Ordr |
| `cdN.allbound.eu` / `cdN.allbound.com` | **Allbound** | Box, LogicMonitor, Global-e, Dialpad, Salt Security, tines |
| `.lb.360ecosystems.com` | **Webinfinity (360insights)** | Zuora |
| `.live.siteforce.com`, or the host is `*.my.site.com` | **Salesforce Experience Cloud** | Datadog, CrowdStrike, DocuSign, Nexthink, Blue Yonder, Employment Hero, Amadeus, Atlassian, NetApp, OpenAI, Hootsuite, SailPoint, Workday |
| `<customer>.magentrixcloud.com` | **Magentrix** | Deep Instinct, Keeper Security, Pigment |
| `<customer>.channeltivity.com` | **Channeltivity** | Tenzai, Yellowbrick Data |
| `<customer>.enterpriseprm.net` | **enterprisePRM** | Fortinet |
| `fallback.eulerapp.com` | **EulerHQ** | Anthropic |
| `fwd.matrixlms.com` | *MatrixLMS — an LMS, not a PRM. See below.* | Axiad |

The Salesforce CNAME embeds the customer's **Org ID** — the `00D...` segment in
`partners.amadeus.com.00d0y000001iskcuao.live.siteforce.com`. That confirms a real Salesforce org,
not a lookalike.

**A CNAME pointing at the company's own infrastructure is positive evidence of self-building** —
this is what lifts the `Likely self-built` verdict off a bare absence:

| CNAME | Reading |
|---|---|
| `lb-ext-02.veeam.com` | Veeam's own load balancer |
| `portal-prod-loadbalancer-…elb.amazonaws.com` | The Trade Desk's own AWS load balancer |

**A generic CDN CNAME proves nothing either way.** 1Password's portal is Zift, but its CNAME is
just `d20kz15r38r85a.cloudfront.net`. When the CNAME is bare CloudFront, Cloudflare, Fastly or
Akamai, fall through to headers and body.

---

## Verified

### Impartner
Confirmed at: GitLab (`partners.gitlab.com`), SentinelOne, Netskope

| Where | Signal |
|---|---|
| CNAME | `.partner-experience.com` **or** `impartner.live` |
| CSP header | `*.prmcdn.io` |
| Body | the string `impartner` (appears multiple times) |

`prmcdn.io` is Impartner's CDN and is the reliable one — the body string can be absent on
white-labelled instances.

**Two CNAME patterns, not one.** `impartner.live` was added 2026-09-09 (Syncari). A table that
only knows `.partner-experience.com` will under-count Impartner, which matters because it is
consistently the most common vendor found.

Ordr confirmed on CNAME **plus the TLS certificate** while the page itself served a generic
"Unavailable" placeholder — the certificate is a usable fallback when the body is uninformative.

---

### Salesforce Experience Cloud
Confirmed at: Datadog, CrowdStrike, Rubrik, DocuSign, Databricks, Nexthink, Blue Yonder,
Employment Hero

| Where | Signal |
|---|---|
| `server` header | `sfdcedge` |
| Host | `*.my.site.com` (Salesforce's own customer-portal domain) |
| Redirect path | final URL ends in `/s/` |
| `set-cookie` | `CookieConsentPolicy`, `LSKey-c$CookieConsentPolicy` |
| Body | `force.com`, `sfdcstatic.com` |

`*.my.site.com` is Salesforce's own domain, so a portal there is **not** off-domain — it passes
Step 3 condition 2. Hootsuite, SailPoint and Workday all sit on it.

**Read the caveat before reporting this.** These fingerprints are identical whether the company
licensed *Salesforce PRM* (a product) or built their own portal on the *Salesforce platform*. You
cannot distinguish them from outside. Verdict is **Built on a CRM platform**, and Notes must say
the licensed-vs-in-house question is unresolvable from public signals.

Databricks matched on the `/s/` path with **no** `sfdcedge` header — a CDN in front of Salesforce
masks the header. Path alone is enough to suspect Salesforce, not enough to confirm it.

---

### Allbound *(also trades as Channelscaler)*
Confirmed at: Box (`partnerportal.box.com`), LogicMonitor, Global-e

| Where | Signal |
|---|---|
| Script / asset hosts | `cdn.allbound.com`, `assets.min.allbound.com`, `fonts.allbound.eu` |
| Customer subdomain | `{customer}.allbound.eu` (e.g. `global-e.allbound.eu`) |
| WordPress theme path | `/wp-content/themes/allbound4.0/` |

Allbound instances run on WordPress — `wp-content` plus an `allbound` host is a solid pair.

**Match the host, never the bare word.** See the substring warning below; `allbound` as a plain
string produces false positives on unrelated platforms.

---

### Zift Solutions / Unifyr
Confirmed at: 1Password (`1password.partners`)

| Where | Signal |
|---|---|
| CSP header | `*.ziftsolutions.com`, `*.ziftone.com`, `*.zift123.com`, `*.ziftmarcom.com` |
| HTML root | `<html ng-app="ziftAppModule">` |
| Meta tag | `<meta name="author" content="Unifyr">` |
| Preconnect | `fontawesome.unifyr.com`, `fontawesome.ziftone.com` |

Mid-rebrand: Zift → Unifyr. Match **either** name. Report as `Zift / Unifyr`.

---

### Webinfinity *(360insights / 360ecosystems)*
Confirmed at: Zuora (`partner.zuora.com`)

| Where | Signal |
|---|---|
| CSP header | `*.webinfinity.com` |

Webinfinity was acquired by 360insights in March 2022 and now trades as 360ecosystems. The
`webinfinity.com` domain is still what appears in the CSP. Report as `Webinfinity (360insights)`.

---

### Kiflo
Confirmed at: vendor application only — not yet seen on a customer portal

| Where | Signal |
|---|---|
| CSP header | `cdn.kiflo.com` |
| Body | `api-private.kiflo.com`, `directory.kiflo.com` |
| `x-powered-by` | `ASP.NET` (weak on its own — needs a `kiflo` host too) |

---

### Magentrix
Confirmed at: Deep Instinct (`portal.deepinstinct.com`), Keeper Security (`partner.keeper.io`),
Pigment (`partnerportal.pigment.com`)

| Where | Signal |
|---|---|
| CNAME | `<customer>.magentrixcloud.com` |
| `set-cookie` | `MAG_STATE_MODULE` |
| URL paths | `/aspx/Login`, `/aspx/Register` |

**The old pattern in this table was wrong.** It listed `*.magentrix.com`, which is Magentrix's
*marketing* domain. Customer tenants live on **`magentrixcloud.com`**, and the string
`magentrix.com` appears on none of the three portals above. Any earlier sweep using the old
pattern silently under-reported Magentrix. See "Marketing domain is not tenant domain" below.

---

### PartnerStack
Confirmed at: Supermetrics, Teleport

| Where | Signal |
|---|---|
| Portal host | `dash.partnerstack.com` — the customer's own site links straight out to it |
| Page title | literally `PartnerStack` |

PartnerStack hosts the portal on its own domain rather than a customer subdomain, so the partner
link on the marketing site is the thing to follow. Off-domain, but a known vendor host, so it
passes Step 3 condition 2.

---

### Channeltivity
Confirmed at: Tenzai (`tenzai.channeltivity.com`), Yellowbrick Data

| Where | Signal |
|---|---|
| Host | `<customer>.channeltivity.com` |
| Path | `/Login` |

Veza also serves genuine Channeltivity assets at `veza.channeltivity.com`, but that host currently
404s with no auth affordances — it fails Step 3 and was correctly left unrecorded. Re-check it.

---

### Mindmatrix
Confirmed at: Traceable (`partners.traceable.ai`)

| Where | Signal |
|---|---|
| TLS certificate | subject `O=MindMatrix` |
| Host pattern | `*.mindmatrix.net` |

The TLS certificate carried the identification here, not the headers or body. Worth remembering
as a signal class in its own right: `openssl s_client` or the cert shown by `curl -v` names the
issuer's organisation even when the page reveals nothing.

---

### PartnerPage
Confirmed at: ContentSquare, Rewind

| Where | Signal |
|---|---|
| Asset hosts | `cdn.partnerpage.io`, `js.partnerpage.io`, `content.partnerpage.io` |
| Body | on-page text about creating an account or logging in to "PartnerPage" |

---

### Introw
Confirmed at: Sedai, and Introw's own portal

| Where | Signal |
|---|---|
| Page title | `Introw Partner Connect` — `Partner Connect` is Introw's named product |
| Host | `partners.<customer>.<tld>` fronting Introw |

---

### enterprisePRM
Confirmed at: Fortinet (`partnerportal.fortinet.com`)

| Where | Signal |
|---|---|
| CNAME | `<customer>.enterpriseprm.net` |
| `set-cookie` | `PPLang`, `PPCsrf` (the `PP` prefix is distinctive) |

---

### Vartopia
Confirmed at: CloudBees (`partners.cloudbees.com`)

| Where | Signal |
|---|---|
| Redirect | 302 to `vartopia.okta.com` |
| Okta app slug | `vartopia_<customer>production` |

Note the interaction with the identity-provider rule below: the Okta host alone would be a
non-answer, but the **app slug names Vartopia**, which makes it a real fingerprint.

---

### JourneyBee
Confirmed at: CoffeeBean Technology

| Where | Signal |
|---|---|
| CSP / asset host | `cdn.journeybee.io` |

---

### EulerHQ *(trades as `tryeuler`)*
Confirmed at: Anthropic (`partnerhub.claude.com`)

| Where | Signal |
|---|---|
| CNAME | `fallback.eulerapp.com` |
| Redirect | 302 to `eulerhq.com` |
| Platform | built on Bubble.io |

---

### PartnerPortal.io
Confirmed at: Siemba

Seen once; capture the exact asset host on the next sighting and fill this row in.

---

## Unverified — patterns only, confirm before trusting

Not yet seen on a live customer portal. A match here is a **lead**, not a confirmation. If you
confirm one, move it up to Verified and note the company.

| Vendor | Likely signals |
|---|---|
| ZINFI | `*.zinfi.com`, `*.zinfi.net` |
| xAmplify | `*.xamplify.com` |
| Microsoft Dynamics / Power Pages | `*.dynamics.com`, `*.powerappsportals.com`, `*.microsoftcrmportals.com` |
| HubSpot | `*.hubspot.com`, `*.hs-sites.com` |
| Oracle PRM | `*.oracleoutsourcing.com`, `*.oraclecloud.com` |

Four rows left this table on 2026-09-09 — PartnerStack, Channeltivity, Magentrix and Mindmatrix
were all confirmed in one 332-company sweep. Read that as encouragement: the remaining five are
probably real patterns that have not yet met a customer, not bad guesses.

---

## Not a PRM — do not report these as the vendor

Two categories that will match a naive grep and produce a wrong answer.

**Identity providers.** These handle the login, not the partner program. A portal fronted by Okta
still has some other PRM behind it — or none.

`*.oktacdn.com` · `okta.com/app/` · `*.auth0.com` · `login.microsoftonline.com` · `*.onelogin.com` ·
`*.pingidentity.com`

Okta's own `partners.okta.com` matches only `*.oktacdn.com`. That is Okta being an identity
company, not Okta being their PRM. Correct verdict there is `Portal found, vendor unclear`.

**Ecosystem and co-sell tools.** Adjacent to PRM, not PRM. Worth noting in Notes — they signal a
real partnership motion — but they are not the answer to "what runs the portal".

`workspan` · `crossbeam` · `reveal.co` · `partnertap` · `partnerfleet`

DocuSign's portal references `workspan` alongside its Salesforce signals. Salesforce Experience
Cloud is the verdict; WorkSpan belongs in Notes.

**Learning platforms.** An LMS behind a "partner" hostname is partner *training*, not partner
management. Axiad's `partner.axiad.com` CNAMEs to `fwd.matrixlms.com` — that is MatrixLMS, and the
honest verdict is `Other` with the LMS named in Notes, not a PRM.

`fwd.matrixlms.com` · `*.docebosaas.com` · `*.talentlms.com`

**Self-hosted app frameworks.** These are evidence *for* `Likely self-built`, never a vendor name.
Deskpro's portal serves `/filament/assets/app.js` and sets `partners_portal_session` — Filament on
Laravel, i.e. something they built.

`/filament/` · `partners_portal_session` · bare Django/Rails/Laravel session cookies

---

## Match hosts, not bare words

Every fingerprint in this file is a **hostname, a header value, or a path** — never a loose
substring. That is deliberate, and the reason is a real bug this table shipped with.

Matching the bare string `allbound` flagged Nexthink, Blue Yonder, and DocuSign as Allbound
customers. All three are plain Salesforce. The match came from a Salesforce Lightning CSS
variable:

```
--agf-squareIconXSmallBoundary: 1.25rem;
```

`sm` + **`allBound`** + `ary`. Three wrong verdicts from one substring, on portals that had
nothing to do with the vendor.

Before adding a row, ask whether the string could occur inside an unrelated word or a framework's
generated CSS. If it could, anchor it to a host (`cdn.allbound.com`) or a path
(`/wp-content/themes/allbound4.0/`) instead. A fingerprint that is merely *usually* right is worse
than no fingerprint, because nobody re-checks a confident answer.

---

## Marketing domain is not tenant domain

The Magentrix row in this table was wrong for as long as the table existed, and nothing caught it,
because a wrong pattern here fails **silently**. It produces `Unclear` — never a wrong vendor — so
no verdict ever looks suspicious enough to re-check.

The cause: the pattern `*.magentrix.com` was derived from the vendor's own website. Their customer
tenants are on `magentrixcloud.com`. Three portals matched the real pattern and none matched the
documented one.

So when adding a row from a vendor's marketing site, **assume the tenant domain differs until you
have seen a live customer portal.** Vendors routinely split them:

| Vendor | Marketing | Customer tenants |
|---|---|---|
| Magentrix | `magentrix.com` | `magentrixcloud.com` |
| Allbound | `allbound.com` | `{customer}.allbound.eu` |
| Impartner | `impartner.com` | `.partner-experience.com`, `impartner.live` |
| EulerHQ | `eulerhq.com` | `fallback.eulerapp.com` |

This is the main reason an Unverified row stays Unverified until a real customer confirms it — the
tier is not about trusting the vendor, it is about not yet knowing where their customers live.

---

## Adding a row

When a portal matches nothing:

1. Capture the unrecognized CSP domains, `server` / `x-powered-by` values, and script-source hosts.
2. Search the odd domain — `prmcdn.io` leads straight to Impartner.
3. Add the row under **Unverified**, or under **Verified** with the company name if you confirmed it.
4. Open a PR, or send it to whoever maintains the repo.

One confirmed row helps everyone who runs this next.
