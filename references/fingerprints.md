# PRM Fingerprints

How to tell what software runs a partner portal, from the raw HTTP response.

**This file is meant to grow.** Vendors rebrand, CDNs change, CSP headers get rewritten. Zift
became Unifyr and both names still appear in the same page source mid-migration. When a portal
matches nothing, record the unmatched signals and add a row here.

Two tiers below. **Verified** rows were confirmed against live portals on 2026-09-08 — the
companies are listed so you can re-check. **Unverified** rows are pattern guesses from vendor
infrastructure and have *not* been seen on a real customer portal. Never report an Unverified
match as `Confirmed`.

---

## Where to look

In priority order — the strongest signals are the ones nobody bothers to hide:

1. **`content-security-policy` header** — lists every domain the page may load from. A PRM vendor's
   CDN is almost always in it. Strongest single signal, and it comes back from a `HEAD` request.
2. **`server` / `x-powered-by` headers** — sometimes names the platform outright.
3. **`set-cookie` names** — platform-specific cookie names leak the stack.
4. **HTML root attributes and meta tags** — `<html ng-app="...">`, `<meta name="author">`.
5. **Script `src` hosts** — the CDN serving the app bundle.
6. **URL path shape after redirect** — e.g. `/s/` for Salesforce. Weakest, but survives when a CDN
   masks the headers.

---

## Verified

### Impartner
Confirmed at: GitLab (`partners.gitlab.com`), SentinelOne, Netskope

| Where | Signal |
|---|---|
| CSP header | `*.prmcdn.io` |
| Body | the string `impartner` (appears multiple times) |

`prmcdn.io` is Impartner's CDN and is the reliable one — the body string can be absent on
white-labelled instances.

---

### Salesforce Experience Cloud
Confirmed at: Datadog, CrowdStrike, Rubrik, DocuSign, Databricks, Nexthink, Blue Yonder,
Employment Hero

| Where | Signal |
|---|---|
| `server` header | `sfdcedge` |
| Redirect path | final URL ends in `/s/` |
| `set-cookie` | `CookieConsentPolicy`, `LSKey-c$CookieConsentPolicy` |
| Body | `force.com`, `sfdcstatic.com` |

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

## Unverified — patterns only, confirm before trusting

Not yet seen on a live customer portal. A match here is a **lead**, not a confirmation. If you
confirm one, move it up to Verified and note the company.

| Vendor | Likely signals |
|---|---|
| PartnerStack | `*.partnerstack.com`, `dash.partnerstack.com` |
| Channeltivity | `*.channeltivity.com` |
| Magentrix | `*.magentrix.com` |
| ZINFI | `*.zinfi.com`, `*.zinfi.net` |
| Mindmatrix | `*.mindmatrix.net` |
| xAmplify | `*.xamplify.com` |
| Microsoft Dynamics / Power Pages | `*.dynamics.com`, `*.powerappsportals.com`, `*.microsoftcrmportals.com` |
| HubSpot | `*.hubspot.com`, `*.hs-sites.com` |
| Oracle PRM | `*.oracleoutsourcing.com`, `*.oraclecloud.com` |

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

## Adding a row

When a portal matches nothing:

1. Capture the unrecognized CSP domains, `server` / `x-powered-by` values, and script-source hosts.
2. Search the odd domain — `prmcdn.io` leads straight to Impartner.
3. Add the row under **Unverified**, or under **Verified** with the company name if you confirmed it.
4. Open a PR, or send it to whoever maintains the repo.

One confirmed row helps everyone who runs this next.
