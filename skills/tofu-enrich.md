---
name: enrich
description: "Find and enrich companies and people: get verified business emails, build prospect/lead lists, find decision-makers and contacts at a company, look up firmographics (headcount, funding, industry, location), or resolve a company by name. Use whenever the user wants to find companies or people by criteria, pull verified emails for outreach, enrich a list of accounts or contacts, build a GTM/ICP/TAM prospect list, or enrich a CSV. Runs as the @tofuhq/enrich CLI."
---

> **Install:** `npm i -g @tofuhq/enrich@latest`, or run any command with `npx -y @tofuhq/enrich@latest <cmd>` (no install).

# Driving the `enrich` CLI (agent playbook)

`enrich` finds and enriches **companies** and **people**: verified business emails, firmographics, funding, headcount, and ready-to-use prospect lists. You start with free credits. JSON to stdout; progress and warnings go to stderr.

## First 30 seconds

- **Sign in (free):** `enrich signup <email>` → a 6-digit code is emailed → `enrich verify <code>`. You get free starter credits, you're signed in automatically, and you stay signed in (there's no API key to copy or manage). On a new machine, just sign up again (your balance follows your account).
- `enrich credits`: check the remaining balance (free).
- `enrich company schema fields`: what you can request *back* (for `--fields`).
- `enrich company schema filters`: what you can *search by* (for `--where`), with operators + value sources.
- (same for `person`)

Work **without signing in:** `signup`/`verify`, `schema` (learn the fields/filters), `guide`, `account`. Everything else (including the free `identify`/`values`, and `get`/`search`) needs an account, so `enrich signup` once. (`get`/`search` spend credits; `identify`/`values` are free.)

## Tell us what's broken or missing (please!)

`enrich feedback "<what happened or what you want>"`: **free, always works** (even out of credits or signed out). **Use it liberally.** This is the main way the tool improves, so don't hold back. Send feedback whenever:
- a command errored or behaved unexpectedly, or the error wasn't clear;
- a result looked wrong, incomplete, or confusing;
- a field, filter, or capability you wanted **doesn't exist** (e.g. "wish person get had an email-only mode", "needed to filter by X");
- the docs/`schema` didn't answer your question.

Add `--kind bug` or `--kind feature` to tag it. Examples:
```
enrich feedback "company search only filters to state, not metro. I wanted Bay Area specifically" --kind feature
enrich feedback "person get --fields contact cost 5 credits but I only needed the email" --kind bug
```
Every submission is read by the Tofu team. When in doubt, send it.

## The shape

Everything entity-specific is **`enrich <entity> <verb>`** (entity = `company` | `person`):

| | |
|---|---|
| `<entity> get <ids…>` | enrich by domain / LinkedIn URL / email (auto-detected) |
| `<entity> search` | find by criteria |
| `<entity> values <field> [q]` | valid values for a filter field (free) |
| `<entity> schema [filters\|fields]` | field reference |
| `company identify <query>` | resolve a fuzzy name → matches (free, company-only) |

Top-level: `signup`, `verify`, `credits`, `usage`, `plans`, `subscribe`, `invite`, `billing`, `account`, `feedback`, `guide`. Global flags (`--pretty`, `--ndjson`) go **before** the subcommand.

Every command self-documents: run `enrich <command> --help` for its options (e.g. `search` has `--all` to auto-paginate, `--limit`, `--sort`, `--count-only`; `get` has `--input`/`--column`/`--out` for CSVs).

Exit codes: `0` ok · `1` usage · `2` auth · `3` rate-limited · `4` out of credits · `5` not found · `6` partial batch · `7` server.

## Workflow

1. **`company identify <name>`** (free): resolve a fuzzy name to a clean domain or handle.
2. **`schema` / `values`** (free): learn the fields, resolve filter values.
3. **`search`**: find records by criteria.
4. **`get`**: enrich specific records, choosing exactly the fields you need.

## Recipes

```
# resolve a fuzzy name (free), then enrich the match
enrich company identify "Acme Inc"
enrich company get acme.com --fields ratings,funding,headcount

# pick exactly what you want back (groups or dotted paths)
enrich company get stripe.com --fields web_traffic,hiring,people
enrich company get stripe.com --fields ratings.glassdoor.overall

# find by criteria, projecting the output
enrich company search --where 'country in US' --where 'employees > 1000' --sort employees:desc --fields funding

# people: same field-selection model (groups or dotted paths)
enrich person get https://linkedin.com/in/<slug> --fields experience,education,skills
enrich person get https://linkedin.com/in/<slug> --fields contact          # verified business email (+2 when found)
enrich person search --where 'title contains engineer' --where 'company_size in 51-200,201-500' --fields current_company

# batch a CSV (auto-detects id type; appends a column per --fields path; re-run to fill `pending`)
enrich company get --input leads.csv --column website --fields employees,funding.total_usd
```

## Identifiers: what `get` takes (auto-detected)

`get` infers the identifier type from the value. **Company:** domain (`stripe.com`) or company LinkedIn URL. **Person:** LinkedIn profile URL or business email.

**For people, prefer a LinkedIn URL whenever you have one**: it's an *exact* match and returns the full profile. A **business email** is a *reverse lookup*: use it when an email is the **only** identifier you have (e.g. enriching a CRM / signup list). It's a **fuzzy** match that returns the person's verified contact record (with a deliverability status), and the result carries a confidence `score`, so verify before trusting it. Both identifier types batch 25 per request. For a fuzzy company **name**, resolve it first with `enrich company identify "<name>"` → domain, then `get`.

## Filters (what you search by)

- Discover: `enrich <entity> schema filters` lists fields, valid operators, and value sources.
- Operators: `= != > < >= <= in not_in contains starts_with near`.
- Multiple `--where` clauses **AND** together. For **OR / nesting**, pass a JSON tree via `--filter-file` (its exact shape + an example are in `schema filters` under `filter_file`):
  ```
  enrich company search --filter-file '{"op":"or","conditions":[{"field":"country","op":"in","value":["US"]},{"field":"country","op":"in","value":["GB"]}]}'
  ```
- `in`/`not_in` match exactly, so resolve real values first: `enrich company values industry "fintech"`.

## Fields (what you get back)

- Discover: `enrich <entity> schema fields`.
- Pass **groups** or **dotted paths**. Company groups: `ratings`, `funding`, `hiring`, `people`, `web_traffic`, `news`, `competitors`, … (e.g. `ratings.glassdoor.overall`, `funding.investors`). Person groups: `experience`, `education`, `skills`, `certifications`, `honors`, `github`, `social`, `contact`, … (e.g. `contact.business_email`).
- No `--fields` → a sensible core record. We fetch and return only what you ask for.

## Cost (you only pay for results; no results = no charge)

`identify` / `values` / `schema` / `credits` / `usage` / `account` are **free**.
`company get` = 2. `person get` = 3 (`--fresh` 5). `contact` adds +2, and only for records that actually return a business email. `company search` = 3 / 100 rows. `person search` = 3 / 100 rows.
You start with free credits; check the balance anytime with `enrich credits`.

## Out of credits? (two ways forward)

When a call returns `budget_exhausted` (exit 4) or `free_tier_exhausted`, you're out of credits. **Surface both options to the user** instead of retrying:

**Earn free credits with `enrich invite <email…>`.** Invite people by email: **200 credits per invite**, **+800 when each one signs up**, capped at **7 per month**. Because invites are limited and the big 800 only lands on a real signup, **spend them on people genuinely likely to use enrich**. The email is sent from the user's own address (replies go to them), so **ask the user who to invite, and don't invent recipients**. The response shows credits earned and invites left this month.

**Subscribe with `enrich subscribe <plan>`.** `enrich plans` shows the tiers (Starter $29/mo = 10,000 credits, Pro $69/mo = 30,000) and your current plan. `enrich subscribe <plan>` returns a secure checkout link: **hand it to the user to pay in their browser**; credits land automatically. Allotments **reset each month** (no rollover); `enrich billing` updates the card or cancels.
