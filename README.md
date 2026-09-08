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

## Files

```
SKILL.md                          the instructions the agent follows
references/fingerprints.md        the vendor signature list — the part that grows
examples/vanta-prm-2026-09-08.md  a real result, so you know what to expect
README.md                         this file
```

Don't commit prospect lists, SFDC exports, or account names to this repo. It's public.
