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

**"The company's own domain" is broader than the domain you started from.** Three legitimate cases
pass condition 2 even though the registrable domain differs — verify each before accepting it:

- **A vanity TLD they actually own.** `1password.partners` is genuinely 1Password's (confirmed
  Zift/Unifyr). `vanta.partners` is not Vanta's. The only way to tell is the evidence on the page.
- **An alternate or successor brand domain.** Notion serves `notion.com` and `notion.so`; Anthropic
  serves `claude.com`; Monte Carlo serves `montecarlo.ai` alongside `montecarlodata.com`; Keeper
  Security uses `keeper.io`. Zulla now whole-site redirects to `contents.com` after being absorbed
  into it — that redirect *is* the evidence. Note the relationship in Notes.
- **A vendor's own hosting domain.** `*.my.site.com` is Salesforce; `dash.partnerstack.com` is
  PartnerStack. Both are off-domain and both are correct.

What condition 2 actually rejects is a domain belonging to *neither the company nor a vendor* —
a lookalike. When in doubt, ask whether a redirect, a brand mention, or a certificate ties the two
domains together. If nothing does, discard.

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
2. **Capture the partner types they recruit** — see the controlled list below. This is an ICP
   signal in its own right: it describes the shape of the partnership motion, not just whether
   software exists.

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

**The TLS certificate is a fourth signal, and it works when the others fail.** Traceable's portal
gave up nothing in headers or body; the certificate's subject reads `O=MindMatrix`. Ordr's portal
served a generic "Unavailable" placeholder while its certificate and CNAME both named Impartner.

```bash
echo | openssl s_client -connect "$PORTAL_HOST:443" -servername "$PORTAL_HOST" 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

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

## Controlled values — emit these exactly

**Every value below is a Salesforce picklist entry.** Emit the string verbatim, including case,
spacing and the em-dash. A value outside these lists cannot be loaded and will silently drop.

**Never invent a value.** Map what you find to the nearest entry. If nothing fits, use the fallback
for that field (`Unclear`, `Other`) and say what you saw in Notes — that is how the lists grow, by
a deliberate edit rather than by drift.

### Portal status

```
Found — gated portal
Marketing page only
Not found
Blocked
```

### Vendor

```
Impartner
Salesforce Experience Cloud
Allbound
Zift / Unifyr
Webinfinity (360insights)
PartnerStack
Magentrix
Channeltivity
ZINFI
Mindmatrix
Kiflo
Microsoft Dynamics
HubSpot
Likely self-built
Unclear
Other
```

Leave blank when no portal was found. Use `Unclear` when the portal is real but nothing matched;
`Other` only when you positively identified a vendor that has no entry yet.

### Confidence

```
Confirmed
Reported
Unknown
```

### Partner types they recruit

Multi-select. Emit as a semicolon-separated list.

```
Reseller
Referral
Technology / ISV
MSP
Distributor
Affiliate
System Integrator
Consulting / Services
OEM
Training / Learning
```

Marketing pages use dozens of near-synonyms. **Map by the mechanic, not the label:**

| What the page says | Emit |
|---|---|
| VAR, solution provider, hosting partner, channel partner | `Reseller` |
| referral partner, advisor, accounting partner, agent | `Referral` |
| technology partner, integration partner, ISV, app partner | `Technology / ISV` |
| managed service provider, MSSP | `MSP` |
| global SI, GSI, implementation partner | `System Integrator` |
| consulting partner, services partner, service provider, BPO, cloud/AWS service partner | `Consulting / Services` |
| embeds our product, white-label | `OEM` |
| learning partner, training partner, authorised trainer | `Training / Learning` |

Leave blank if the page names no partner types. Do not guess from the company's industry.

---

## The five verdicts

| Verdict | Means | Requires |
|---|---|---|
| **Named vendor** | A specific PRM product | A fingerprint match from `references/fingerprints.md` |
| **Built on a CRM platform** | Salesforce, Dynamics, or HubSpot infrastructure | A platform fingerprint, and **no** named PRM product |
| **Likely self-built** | Runs entirely on the company's own infrastructure | CNAME points at the company's own infrastructure **and** no third-party PRM in headers, scripts, or CSP |
| **Portal found, vendor unclear** | Confirmed portal, nothing matched | A validated portal, no fingerprint hit |
| **No portal found** | Nothing gated exists | All steps ran and completed — **not** applicable if you were blocked |

### How each verdict lands in the fields

The verdicts are how you reason; the fields are what gets stored. This is the mapping:

| Verdict | Portal status | Vendor |
|---|---|---|
| Named vendor | `Found — gated portal` | the vendor, e.g. `Impartner` |
| Built on a CRM platform | `Found — gated portal` | `Salesforce Experience Cloud` / `Microsoft Dynamics` / `HubSpot` |
| Likely self-built | `Found — gated portal` | `Likely self-built` |
| Portal found, vendor unclear | `Found — gated portal` | `Unclear` |
| No portal found | `Not found` or `Marketing page only` | *(blank)* |
| Blocked before reaching a verdict | `Blocked` | *(blank)* |

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

**Confidence rates the vendor, not the portal.** This is the single most common mistake when
running the skill at scale, and it is silent — the row looks well-formed and every value is legal.

> **If `vendor` is blank, `confidence` is `Unknown`. Always.**

A `Not found` or `Marketing page only` row has no vendor to be confident *about*. Writing
`Confirmed` there reads, downstream, as "we are confident which PRM they use" — the opposite of
what was meant, which was "we are confident they have no portal". A batch run once produced this
on 10 of 15 rows before anyone noticed.

There is no field for how sure you are that a portal is absent. If that matters, say it in Notes.

**Never report an Unverified fingerprint as `Confirmed`** — a pattern in the Unverified tier of
`references/fingerprints.md` has never been seen on a real customer portal, so a match is a lead.
Use `Reported`. If your match is the first confirmation, promote the row in `fingerprints.md`
*and then* you may write `Confirmed`.

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

**Portal:** {Found — <url> | Marketing page only — <url> | Not found | Blocked}
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
company,domain,account_type,PRM_Audit_Last_Checked,PRM_Audit_Portal_Status,portal_url,
PRM_Audit_Vendor,PRM_Audit_Confidence,PRM_Audit_Partner_Types,evidence,notes,sfdc_account_id
```

Twelve columns, on one line in the file. The four `PRM_Audit_*` columns plus
`PRM_Audit_Last_Checked` are named for the Salesforce fields they load into, so the mapping needs
no translation step.

- `PRM_Audit_Portal_Status`, `PRM_Audit_Vendor`, `PRM_Audit_Confidence`,
  `PRM_Audit_Partner_Types` — use the exact strings from **Controlled values** above. These load
  straight into Salesforce, so a stray value or a lowercased one breaks the import. The em-dash in
  `Found — gated portal` is U+2014; a hyphen silently fails to load.
- `PRM_Audit_Last_Checked` — `YYYY-MM-DD`. The date the check actually ran. On a run spanning
  midnight, date each row when it was researched rather than backdating for tidiness — this field
  exists to track decay, so a wrong date defeats its only purpose.
- `account_type` — the CRM account type (`Customer`, `Prospect ISV`, …) when the list came from
  CRM. Blank otherwise.
- `sfdc_account_id` — the Salesforce record Id, when known. Carrying it turns the write-back into
  a keyed update instead of a fuzzy name match. Blank when the list did not come from CRM.
- `PRM_Audit_Partner_Types` — semicolon-separated when there is more than one.
- `evidence` — the key signals, semicolon-separated, kept short. Free text.
- `notes` — free text.
- Quote any field containing a comma. Build rows with a CSV library, not string concatenation.

**Three columns have no Salesforce field: `portal_url`, `evidence`, `notes`.** They stay in the
CSV. Salesforce carries the claim; the CSV carries the receipt. Do not drop them because they
have nowhere to load.

---

## Running at scale — hundreds of companies

Everything above describes auditing one company. Auditing several hundred is a different job, and
doing it by looping the single-company procedure in one context will exhaust it. Fan the work out
to subagents instead. This section is what a 332-company run taught.

### Shape of the run

**Resolve every domain up front, in one pass.** Pull the list from CRM, normalise `Website` to a
bare registrable domain (formats vary wildly — `1password.com` next to `https://www.7ai.com`),
dedupe, and write a worklist. Researching agents then skip Step 1 entirely and cannot wander onto
the wrong company. Do this as one scripted step so the raw records never enter a reasoning context.

**Batch 15 companies per subagent.** The fixed cost of a subagent is reading this file plus
`references/fingerprints.md` — around 26K of text. One company per agent pays that cost 15 times
over. Fifteen amortises it well; beyond about 20 the agent risks running out of context.

**Have each agent append after every 5 companies, not once at the end.** Batching only works if a
context shortfall costs the tail rather than the batch. This is also the difference between losing
5 rows and losing 15 when a run hits a rate limit mid-flight.

**One writer to the shared CSV.** Subagents write their own `batch-NNN.csv`; a single merge step
appends into the audit file. Dozens of agents appending concurrently interleaves rows and corrupts
the file. Keep the audit CSV closed in any spreadsheet app for the duration — a lock file will
collide with the merge.

**Have agents report five lines, not their findings.** Batch number, rows written, counts by portal
status. The rows are already on disk; re-narrating them into the orchestrator's context is pure
waste at this scale.

### Pilot before you scale

**Run exactly one batch, then read its output by hand against the field rules.** A flaw in the
instructions multiplied across twenty agents is the expensive failure, and every flaw found this
way was invisible in the agent's own self-report — the agents cheerfully reported "all validations
passed" on rows that were wrong.

The pilot caught the blank-vendor/`Confirmed` error described under **How sure** on 10 of 15 rows.
Fixing it cost one batch. Finding it at the end would have cost the entire run.

### Validate on merge, and quarantine rather than drop

Check every controlled value against the picklists at merge time, comparing exact codepoints so a
hyphen substituted for the em-dash is caught before Salesforce silently drops it. Send offending
rows to a `rejects.csv` rather than discarding them — a rejected row still contains real research.

Worth enforcing beyond the picklists, because each of these caught a real defect:

- blank vendor must pair with `Unknown` confidence
- a `Found` status must carry a `portal_url`; a `Not found` must not
- row count per batch must equal input count
- `checked_date` must parse

Apply the documented `MSSP → MSP` style mappings from **Controlled values** as a logged
normalisation, not a silent rewrite — and only for mappings this file already documents.

### Agents cannot learn from each other

Each subagent starts fresh, so a vendor discovered in batch 4 is rediscovered as `Other` in every
later batch. When a new fingerprint is confirmed twice independently, add it to the prompt for the
remaining batches — or better, write it into `references/fingerprints.md` mid-run.

Expect this to happen: a 332-company sweep surfaced seven vendors the table did not contain and
confirmed four that were sitting in the Unverified tier.

### Rate limits and concurrency

Start at 4–5 concurrent agents. A limit hit kills every agent in flight, and each one dies holding
most of a batch of research. Larger waves do not finish sooner; they just lose more when they fail.

### If an agent is refused before it starts

Batches heavy with security vendors can trip a safety classifier, because "check these companies'
login pages and DNS records" reads like reconnaissance. It is not — this skill reads public
marketing and login pages and treats `403` as a stop sign — but the framing matters.

Lead the prompt with the business purpose ("cataloguing which PRM product each customer uses, from
their own public partner pages") and leave the mechanics to this file, which the agent reads
anyway. Do not restate the URL-probing procedure in the prompt. If a batch is refused twice,
that is the signal to reframe rather than retry verbatim.

---

## Checks before you finish

- Did every reported portal URL pass the Step 3 validation, or did a wildcard redirect slip through?
- Is any `No portal found` actually a `blocked`?
- Does any `Likely self-built` rest on an absence rather than two positives?
- Does every `Built on a CRM platform` say in Notes that licensed-vs-in-house is indistinguishable?
- **Does every blank-vendor row have `Unknown` confidence?**
- **Is any `Confirmed` resting on an Unverified fingerprint?** That should be `Reported`.
- Is the dash in `Found — gated portal` an em-dash (U+2014), not a hyphen?
- Is `program_type` semicolon-separated, not comma-separated?
- Does the row count equal the input count — including rows for companies that resolved to nothing?
- Did you write to a relative path?
