---
name: prm-scrape
description: "Check whether a company has a partner portal, and identify what PRM software runs it. Use for partner-portal checks, PRM incumbent research, or channel-program signal gathering on a named company or list of companies."
---

# PRM Scrape

Answer two questions about a company:

1. **Do they have a partner portal?**
2. **What software runs it** — a named PRM vendor, a CRM platform, or something they built themselves?

Output is a short audit doc (one company) or a CSV row (several companies). Every claim carries
its evidence, so a rep can check the receipt before repeating it on a call.

---

## Rules — read these first

These are not style preferences. Every request this skill makes hits another company's
infrastructure from your employer's IP, attached to your employer's name.

**1. Never log in. Never submit a form.**
Read pages. Do not type into fields, do not click submit, do not try credentials, do not use
"forgot password" to test whether an email exists. This is the line between research and
unauthorized access. An eager agent "just checking if the portal is real" is one bad instruction
away from crossing it. Fetching a login page is fine. Interacting with it is not.

**2. `403` and `429` are stop signs, not obstacles.**
When a site blocks you, record `blocked` and move to the web-search step. Do **not** retry with a
different user agent, add delays and try again, rotate IPs, or otherwise work around the block.
A tool that evades bot protection is a different and much worse thing than a tool that reads
public pages.

**3. Cap at 15 requests per company.**
Eight URL guesses, a marketing page, a couple of redirects — that's the budget. Without a cap, an
agent that can't find a portal will invent candidate URLs forever, and 30 companies of that starts
to look like scanning.

**4. Stay at the front door.**
Fetch the portal's landing or login page to fingerprint it. Do not crawl deeper. Do not probe
`/admin`, `/api`, `/.well-known`, `/robots.txt`-listed paths, or enumerate directories.

**Never write to an absolute path.** Write output relative to the current working folder. This
skill runs on other people's machines.

---

## Input

A company name, a domain, or a list of either.

- **One company** → write an audit doc.
- **Several companies** → append rows to a CSV.
- The user can override by saying "as a doc" or "as a sheet".

---

## Step 1 — Resolve the domain

If given a company name, find its primary domain (web search if needed). If the name is ambiguous
— multiple real companies share it — stop and ask which one. Do not guess. A confident audit of the
wrong company is worse than no audit.

If given a domain already, skip this step.

---

## Step 2 — Guess the obvious URLs

Probe these eight in parallel. Follow redirects, record the final URL and status code:

```
https://partners.{domain}
https://partner.{domain}
https://{domain}/partners
https://{domain}/partner-portal
https://partnerportal.{domain}
https://{brand}.partners          # brand = domain minus TLD, e.g. 1password.partners
https://portal.{domain}
https://connect.{domain}
```

```bash
for u in <the eight URLs>; do
  curl -s -o /dev/null -w '%{http_code} %{url_effective}\n' -L --max-time 8 \
    -A 'Mozilla/5.0' "$u"
done
```

This finds a portal for roughly 7 of 8 companies. Vanity domains are real —
`1password.partners` is not a subdomain of `1password.com`.

---

## Step 3 — Validate every hit

**A `200` does not mean a portal exists.** Many companies have wildcard DNS that silently sends
every unknown subdomain to their marketing homepage. In testing, MongoDB and Twilio each produced
three `200` responses that were all mirages.

A candidate counts as a **gated portal** only if **all three** are true:

- The final URL is **not** the marketing site (not `www.{domain}` or `{domain}/` homepage), **and**
- The final URL's registrable domain is **either the company's own domain or a known PRM vendor's
  domain** from `references/fingerprints.md`, **and**
- The page shows **auth affordances** — a password field, "Sign in" / "Log in" / "Register",
  or a redirect to an SSO provider.

**The second condition is the one that is easy to forget, and it fails loudly.** Testing
`vanta.partners` returned a live `200` with a login form — at `www.vantapartners.io`, which is a
*different company* with a similar name. The vanity-TLD guess in Step 2 is the usual culprit: it is
the one candidate that isn't anchored to the company's real domain. If the final domain belongs to
neither the company nor a known vendor, discard it. Do not report it with a caveat.

**Detect auth affordances in the raw HTML, not in rendered or converted text.** These pages are
SPA shells whose visible text is nearly empty, but the login markup survives in scripts, JSON
blobs, and attributes:

```bash
curl -sL --max-time 12 -A 'Mozilla/5.0' "$URL" | head -c 60000 \
  | grep -icE 'type="password"|sign ?in|log ?in|register'
```

**A `<title>` naming the portal also counts.** Some portals gate everything behind JavaScript and
expose no auth strings at all in the shell — Veeam's returns `<title>Veeam ProPartner Portal</title>`
and The Trade Desk's returns `<title>Partner Portal</title>`, both with zero password or sign-in
markup. A title matching `partner portal` / `partner login` is sufficient on its own.

Any count of 1 or more passes. The count is not a confidence score — confirmed portals scored
anywhere from 1 (GitLab, Vanta) to 75 (Box). Zero across a page that also has no auth-shaped
redirect means it is not a gated portal.

If the final URL is a marketing page describing the partner program with no login wall, that is a
**marketing page**, not a portal. Record it as such — it is a distinct and useful state, not a
failure.

Never report a guessed URL that has not passed this check.

---

## Step 4 — Read the marketing site

Fetch `{domain}/partners` (or whatever the homepage nav/footer links to as "Partners").

Two jobs here:

1. **Find the portal link** if Step 2 missed it. Look for "Partner Login", "Partner Sign In",
   "Portal". This catches vanity domains and odd naming that guessing cannot.
   **Do not skip this step when Step 2 found nothing.** Veeam's portal is at
   `propartner.veeam.com` — none of the eight guesses reach it, but the link sits in plain sight
   on `veeam.com/partners`. Guessing has a fixed vocabulary; companies do not.
2. **Capture the program type** — which kinds of partners they run. Record whichever apply:
   `reseller` · `referral` · `technology / ISV` · `MSP` · `distributor` · `affiliate` ·
   `system integrator`. This is an ICP signal in its own right: it describes the shape of the
   partnership motion, not just whether software exists.

---

## Step 5 — Web search fallback

Only if Steps 2–4 found nothing, or the site blocked you.

Search `"{company} partner portal login"` and `"{company}" PRM OR "partner portal" vendor`.

Third-party sources — press releases ("Acme selects Impartner"), a vendor's customer logo wall,
review sites — can name the vendor. That evidence is **Reported**, never **Confirmed**. Label it
honestly.

**Check that every result is the right company.** Search is the one rung that can quietly hand you
a different business. Searching for Ramp's (`ramp.com`, spend management) partner portal returns
detailed documentation for *Ramp Network* (`rampnetwork.com`, crypto payments) — a real partner
portal belonging to an unrelated company. Confirm the domain in the result matches the domain from
Step 1 before believing anything it says.

---

## Step 6 — Fingerprint the portal

Read `references/fingerprints.md` and match against it.

**Start with DNS.** One lookup, no HTTP request, and it survives WAFs and CDNs that block or mask
everything else:

```bash
dig +short "$PORTAL_HOST" CNAME
```

`*.partner-experience.com` is Impartner. `cdN.allbound.eu` is Allbound. `*.live.siteforce.com` is
Salesforce. `*.lb.360ecosystems.com` is Webinfinity. Amadeus's portal is behind Imperva and returns
an empty body to automated requests — its CNAME names Salesforce anyway. A CNAME pointing at the
company's own load balancer is positive evidence of self-building.

A bare CloudFront/Cloudflare/Fastly CNAME means nothing either way — fall through to the headers.

Then pull headers and body. The vendor name is almost never visible on the page — it lives in
response headers, CSP directives, script sources, and HTML attributes:

```bash
curl -sSIL --max-time 10 -A 'Mozilla/5.0' "$PORTAL_URL" \
  | grep -iE '^(server|x-powered-by|content-security-policy|set-cookie|location)'

curl -sL --max-time 12 -A 'Mozilla/5.0' "$PORTAL_URL" | head -c 40000
```

**Do not use a markdown-converting fetch tool for this step.** These portals are JavaScript apps
with near-empty HTML shells — GitLab's renders 281 characters of visible text, Datadog's renders
one. Converting to markdown throws away the headers, meta tags, and script sources that carry the
entire signal. Raw HTTP is the only method that works.

**A rendered browser does not help either.** Rendering shows you a login form. It does not show
you "Impartner" — that string is in a CSP header, not on screen.

### When signals conflict

A page can match several fingerprints at once. Apply this precedence:

1. **A named PRM product wins** over a CRM platform. Allbound-on-Salesforce is Allbound.
2. **A CRM platform wins** over nothing.
3. Mention the other signals in Notes rather than dropping them.

**Before applying precedence, check the match is real.** Match hostnames and header values, never
bare substrings — `references/fingerprints.md` documents a case where the loose string `allbound`
produced three wrong verdicts by matching inside a Salesforce CSS variable named
`--agf-squareIconXSmallBoundary`.

---

## The five verdicts

| Verdict | Means | Requires |
|---|---|---|
| **Named vendor** | A specific PRM product | A fingerprint match from `references/fingerprints.md` |
| **Built on a CRM platform** | Salesforce, Dynamics, or HubSpot infrastructure | A platform fingerprint, and **no** named PRM product |
| **Likely self-built** | Runs entirely on the company's own infrastructure | CNAME points at the company's own infrastructure **and** no third-party PRM in headers, scripts, or CSP |
| **Portal found, vendor unclear** | Confirmed portal, nothing matched | A validated portal, no fingerprint hit |
| **No portal found** | Nothing gated exists | All steps ran and completed — **not** applicable if you were blocked |

Three things that go wrong here, and the rules that prevent them:

**"Built on a CRM platform" is genuinely ambiguous, and the doc must say so.**
Salesforce Experience Cloud produces byte-identical fingerprints whether the company licensed
Salesforce PRM or built their own portal on the Salesforce platform. You cannot tell from outside.
Write that limitation into the Notes every time — do not silently pick one.

**"Likely self-built" needs two positives, not one absence.**
Absence of a fingerprint is not evidence of self-building. The DNS `CNAME` supplies the missing
positive: Veeam's portal points at `lb-ext-02.veeam.com` and The Trade Desk's at
`portal-prod-loadbalancer-…elb.amazonaws.com` — both the company's own load balancers. That is
evidence. A company that returned `403` and never resolved a CNAME meets neither condition — that
is `Unknown`, not self-built.

**Blocked is not the same as absent.**
If a site returned `403` or `429`, the verdict is `Portal found, vendor unclear` or a explicit
`Blocked` note — never `No portal found`. Snowflake and HashiCorp both block automated requests
and both have real partner programs.

---

## How sure — the evidence ladder

Three labels, in plain language:

- **Confirmed** — the vendor's own fingerprint was found on the live portal.
  *"Their login page loads code from Impartner's servers."*
- **Reported** — a third party says so: press release, vendor customer list, review site.
  Believable, not proven.
- **Unknown** — blocked, or nothing matched. Says nothing either way.

---

## When nothing matches — grow the table

If the portal is real but no fingerprint matched, **report the unmatched signals you saw**: any
unrecognized domains in the CSP header, unusual `server` or `x-powered-by` values, unfamiliar
script sources or CDN hosts.

Put them in the Notes. This turns every miss into a submittable fingerprint instead of a dead end —
it is how `references/fingerprints.md` grows. Send them to whoever maintains the repo.

---

## Output

Same fields either way. Pick the block that matches the mode.

### Mode A — audit doc (one company)

Write to `./prm-audits/{company-kebab-case}-prm-{YYYY-MM-DD}.md`, creating the folder if needed.

```md
# {Company} — Partner Portal Audit
**Checked:** {YYYY-MM-DD} · **Domain:** {domain}

**Portal:** {Found — <url> | Marketing page only — <url> | Not found}
**Running on:** {vendor name | CRM platform name | Likely self-built | Unclear}
**How sure:** {Confirmed | Reported | Unknown}
**Program type:** {reseller, referral, technology/ISV, MSP, ...}

## Evidence
- {each signal, in plain language, with the raw string in backticks}
- e.g. Login page loads code from Impartner's CDN (`*.prmcdn.io` in security header)
- e.g. The word "Impartner" appears 3× in the page source
- e.g. Login wall present — email + password fields, no public access

## What we checked
- {each step run, and its result — including failures}
- e.g. Guessed 8 common portal URLs → hit on partners.gitlab.com
- e.g. Read gitlab.com/partners for program details
- e.g. Site returned 403 (bot protection) — fell back to web search

## Notes
- {ambiguity, second portals, reasons to doubt the verdict, unmatched signals}
```

### Mode B — CSV row (several companies)

Append to `./prm-audit.csv`, creating it with a header row if it does not exist. One row per
company. Appending across sessions is intended — Monday's 30 companies and Friday's 20 land in
one sheet.

Columns, in order:

```
company,domain,checked_date,portal_found,portal_url,vendor,confidence,program_type,evidence,notes
```

- `portal_found` — `yes` / `marketing_only` / `no` / `blocked`
- `vendor` — the product name, the platform name, `self-built`, or `unclear`
- `confidence` — `confirmed` / `reported` / `unknown`
- `evidence` — the key signals, semicolon-separated, kept short
- Quote any field containing a comma.

---

## Checks before you finish

- Did every reported portal URL pass the Step 3 validation, or did a wildcard redirect slip through?
- Is any `No portal found` actually a `blocked`?
- Does any `Likely self-built` rest on an absence rather than two positives?
- Does every `Built on a CRM platform` say in Notes that licensed-vs-in-house is indistinguishable?
- Did you write to a relative path?
