# PRM Scrape

A skill that checks whether a company has a **partner portal**, and figures out **what software
runs it** — Impartner, Salesforce, something they built themselves, or nothing at all.

You give it a company. It gives you a short doc with the answer and the receipts.

---

## Why

We have no reliable way to tell whether a prospect is running a partner program on real software,
on a spreadsheet, or not at all. That's a signal worth having before a call — and it's sitting in
public on their website, if you know where to look.

The trick: a partner portal's login page loads code from its vendor's servers, and that shows up in
the raw web response. GitLab's portal quietly loads from Impartner's CDN. 1Password's loads from
Zift. You can't see it on the page — but it's there in the response, and this skill reads it.

---

## Install

### Option A — from GitHub (recommended)

Keeps you on the current fingerprint list. Vendors rebrand and change infrastructure, so an old
copy quietly goes stale.

```bash
git clone https://github.com/atlau101/PRM-Scrape-Skill-.git ~/.claude/skills/prm-scrape
```

To update later:

```bash
cd ~/.claude/skills/prm-scrape && git pull
```

### Option B — download the folder

If you'd rather not touch git: on the GitHub page, click **Code → Download ZIP**, unzip it, and
move the folder into your skills directory:

- **Claude Code:** `~/.claude/skills/`
- **Cowork:** add it as a skill in your plugin/skills settings

Rename the folder to `prm-scrape`. That's it.

Downside of Option B: your copy is frozen. Re-download every couple of months, or when a result
looks wrong.

---

## Use

Just ask, in plain language:

```
Run prm-scrape on Snowflake
```

```
Check whether Databricks, Rubrik, and Netskope have partner portals
```

**One company** → you get a doc at `./prm-audits/{company}-prm-{date}.md`
**Several companies** → you get rows appended to `./prm-audit.csv`, which opens in Excel

Override the format by saying "as a sheet" or "as a doc".

---

## What you get back

```
Portal:       Found — https://partners.gitlab.com
Running on:   Impartner
How sure:     Confirmed
Program type: Reseller + technology/ISV partners
```

Plus the evidence behind it, and a list of what was checked — including anything that failed.

### The five possible answers

| Answer | What it means for you |
|---|---|
| **Named vendor** | They're running a real PRM. Displacement conversation. |
| **Built on a CRM platform** | Bolted onto Salesforce or Dynamics. Often a painful setup. |
| **Likely self-built** | They built it themselves and maintain it themselves. |
| **Portal found, vendor unclear** | Portal is real, we couldn't identify the software. |
| **No portal found** | No gated portal exists. Check if there's a program page. |

Don't read "No portal found" as "no partner program." A company with a partner program page and no
portal is running partnerships on email and spreadsheets — that's a *strong* signal, not a null
result.

### How sure

- **Confirmed** — we found the vendor's fingerprint on their live portal. Safe to say out loud.
- **Reported** — a press release or third-party source says so. Believable, not proven.
- **Unknown** — we got blocked, or nothing matched. Says nothing either way.

**"Unknown" is not "no."** Some sites block automated requests — Snowflake and HashiCorp both do,
and both have real partner programs. The doc tells you when that happened.

---

## When it says "vendor unclear"

That usually means we hit a PRM vendor that isn't in our list yet.

The doc will show the unrecognized signals it found — odd domain names in the page's security
settings, unfamiliar servers. **Send those to Andrew** (or open a PR against
`references/fingerprints.md`). One confirmed addition helps everyone who runs this afterward.

---

## What it will not do

By design, and deliberately:

- **It never logs in or fills in a form.** It reads public pages. That's the line between research
  and unauthorized access, and it doesn't get crossed.
- **It stops when a site blocks it.** No retrying with disguises, no working around bot protection.
- **It makes at most 15 requests per company** and doesn't crawl past the front page.

If you need something past that line, do it by hand as yourself — don't ask the skill to.

---

## Discovery ladder

How the skill actually looks for a portal. Rungs 1–4 are a search: each one runs only if the one
before it came up empty. Rung 5 is not a fallback — validation and fingerprinting run on every
candidate the search turns up.

**Rung 1 — Get the real domain.**
Company name in, canonical domain out (`abcdefg.com`). If the name is ambiguous and more than one
real company matches, the skill stops and asks rather than guessing. A confident audit of the wrong
company is worse than no audit.

**Rung 2 — Guess the obvious URLs.**
Eight candidates in parallel, about two seconds: `partners.X.com`, `partner.X.com`,
`X.com/partners`, `X.com/partner-portal`, `partnerportal.X.com`, `X.partners`, `portal.X.com`,
`connect.X.com`. This found a portal for 7 of the first 8 companies tested.

**Rung 3 — Read the partner marketing page.**
Fetch `X.com/partners` (or whatever the nav and footer link to), and look for "Partner Login",
"Sign In", "Portal".

This rung does two jobs, and **it does not get skipped just because Rung 2 found nothing.** Veeam's
portal lives at `propartner.veeam.com` — no guess in Rung 2 reaches it, but the link sits in plain
sight on `veeam.com/partners`. Guessing has a fixed vocabulary; companies do not.

Its second job is to record the **program type** — reseller, referral, technology/ISV, MSP,
distributor, affiliate, system integrator. That describes the shape of the partnership motion, and
it is a useful signal on its own, separate from which software they run.

**Rung 4 — Web search.**
`"<company> partner portal login"`. Catches sites that blocked the earlier rungs, and anything the
first three missed. Anything found this way is labelled **Reported**, never **Confirmed** — a press
release is weaker evidence than the live portal.

**Rung 5 — Validate, then fingerprint.**
Every candidate gets checked before it is believed. It counts as a real gated portal only if all
three hold:

1. The final URL is **not** the marketing site.
2. The final domain belongs to **the company or a known PRM vendor**.
3. The page shows a **login wall** — password field, "Sign in", an SSO redirect, or a `<title>`
   naming it a partner portal.

Both conditions 1 and 2 exist because of real false positives. MongoDB and Twilio each returned
three live `200`s that were just wildcard DNS pointing at their marketing homepage. And
`vanta.partners` returned a working login page belonging to `vantapartners.io` — **a different
company with a similar name.**

Only after a candidate passes does the skill fingerprint it against `references/fingerprints.md`
to identify the vendor.

### When a site blocks it

A `403` or `429` means the site refuses automated requests. The skill records that and moves to
Rung 4 — it does not retry in disguise. The result is reported as `blocked`, which is **not** the
same as "no portal found". Snowflake and HashiCorp both block, and both have real partner programs.

---

## Files

```
SKILL.md                          the instructions the agent follows
references/fingerprints.md        the vendor signature list — the part that grows
examples/vanta-prm-2026-09-08.md  a real result, so you know what to expect
README.md                         this file
```

Don't commit prospect lists, SFDC exports, or account names to this repo. It's public.


