# Registrum

**UK Companies House data, parsed — with the parts we couldn't parse labelled as such.**

[![API status](https://uptime.betterstack.com/status-badges/v1/monitor/4158135.svg)](https://status.registrum.co.uk)
[![npm](https://img.shields.io/npm/v/%40registrum%2Fmcp?label=%40registrum%2Fmcp)](https://www.npmjs.com/package/@registrum/mcp)
[![Docs](https://img.shields.io/badge/docs-api.registrum.co.uk-0b7285)](https://api.registrum.co.uk/docs)
[![Licence](https://img.shields.io/badge/examples-MIT-green)](https://github.com/RegistrumUK/registrum-mcp-examples)

Companies House publishes the UK register for free, and it is genuinely good data. The
difficulty is never getting a response — it's that a working integration against 20 companies
often behaves very differently across 5.7 million. Accounts arrive as iXBRL *or* as a scanned
PDF. Officer lists paginate by offset, not page number. Auth is HTTP Basic with the key as the
*username*. Identity-verification state under ECCTA is not a boolean.

**These repos are where we put the parts of that worth sharing regardless of whether you ever
become a customer.** Two of them don't mention our API until the last section, on purpose.

---

## Try it right now, without an account

Point any MCP client at this URL. No signup, no API key, no install:

```
https://registrum.co.uk/api/mcp
```

```json
{ "mcpServers": { "registrum": { "url": "https://registrum.co.uk/api/mcp" } } }
```

Then ask it *"Who ultimately owns Rolls-Royce Holdings?"* and it will trace the ownership chain
through the corporate PSCs to the natural persons at the top.

The anonymous tier is generous on everyday lookups and deliberately small on the expensive ones
— a couple of ownership traces and sets of parsed accounts a day, which is enough to judge the
data before you decide anything. Caps announce themselves; nothing silently degrades.

---

## Start here

| Repo | What it is | Use it when |
|---|---|---|
| [**companies-house-api-python-starter**](https://github.com/RegistrumUK/companies-house-api-python-starter) | One-file Python client for the **official free CH API** | You want to call Companies House directly and not fall into the usual four traps |
| [**companies-house-api-node-starter**](https://github.com/RegistrumUK/companies-house-api-node-starter) | Same, in Node, using built-in `fetch` | As above, on the JS side |
| [**registrum-mcp-examples**](https://github.com/RegistrumUK/registrum-mcp-examples) | Prompt cookbook + runnable MCP client scripts | You're wiring UK company data into Claude, Cursor, or your own agent |
| [**@registrum/mcp**](https://github.com/vdmeu/registrum-mcp) | The MCP server itself (npm, MIT) | You want the tools in your agent — hosted (no key) or self-hosted with your own |

The two starter kits are deliberately **honest wrappers around the free official API**. They
hide nothing and they don't need us. If the raw register is all you need, use them and go.

---

## What we add on top

We run the parsing so you don't carry the parsing risk. The interesting part isn't that we
extract fields — it's that we tell you **how well** we did on every single response:

```jsonc
// GET /v1/company/03824658/financials
{
  "available": true,
  "period_end": "2024-12-31",
  "currency": "GBP",
  "data_quality": {
    "source": "ixbrl",
    "completeness": 0.5,           // <- we extracted 10 of 20 attempted fields
    "fields_attempted": 20,
    "fields_extracted": 10,
    "missing_fields": ["turnover", "gross_profit", "operating_profit", "..."],
    "taxonomy_version": "FRS-102/2025-01-01",
    "filed_on": "2025-09-16"
  },
  "profit_and_loss": {
    "turnover": null,                                    // absent from the filing
    "profit_after_tax": { "current": 2173626, "prior": 2344822 }
  },
  "balance_sheet": {
    "net_assets": { "current": 3853781, "prior": 3461016 },
    "cash":       { "current": 3847973, "prior": 4323890 }
  }
}
```

That's a real, unedited response. Two things it demonstrates:

- **`null` means "not in the filing", not "we gave up".** Small-company accounts frequently
  omit turnover entirely. Silently returning `0` there is how screening models end up wrong.
- **Every figure is a `{ current, prior }` pair**, because a single year of a UK filing is
  rarely enough to conclude anything.

When accounts are a flat or scanned PDF with no structured data behind them, you get
`available: false` and an explicit `unavailable_reason` — not an empty object you have to
interpret. That happens more than people expect: **Tesco PLC's own latest accounts return
`image_pdf`.**

### Ownership, resolved rather than returned

A PSC register tells you the immediate controlling parties. When one of those is itself a
company — which, for anything with a corporate structure, it usually is — the register hands
you another company number and stops. Walking that up to real people is the actual job.

`/company/{n}/psc/chain` does that traversal and returns a **tree**, not a flat list, with each
branch marked as terminal and a reason why it terminated: a natural person, a foreign entity
outside the UK register, or a depth limit you set. You can also resolve a company by name
first — `/search` — so a list of counterparty names is a usable input, not a blocker.

### The counter-argument to "we don't refine anything"

Some tools deliberately return raw filing bytes and call unrefined data a feature — no parser
to trust, no opinion imposed. It's a fair position, and if you're building your own extraction
pipeline it's the right one.

But it doesn't remove the parsing problem; it relocates it to you, permanently, including every
taxonomy revision from here on. The honest question isn't *refined vs raw*, it's **who carries
the parsing risk, and do they tell you when it fails.** Our answer is: we do, and yes —
`completeness`, `missing_fields` and `taxonomy_version` ship on every response so you can set
your own threshold and reject ours.

---

## ECCTA identity verification

From **18 November 2026**, UK directors and PSCs must verify their identity with Companies
House. The compliance endpoint reports register state per company:

```jsonc
// GET /v1/company/00445790/compliance   (Tesco PLC)
{
  "directors_total": 10,
  "directors_verified": 3,
  "directors_pending": 7,
  "directors_overdue": 0,
  "verification_rate": 0.3,
  "verification_risk": "partial",
  "unverified_persons": [ /* name, role, status, deadline */ ]
}
```

Note that `pending` and `overdue` are **separate counters**. Treating verification as a
verified/unverified boolean — or folding "pending" into "overdue" — will misreport a large
share of the register during the transition. That distinction is the single most common
mistake we see in this area, and it is worth getting right whether or not you use us for it.

---

## Why you might trust the parsing

| | |
|---|---|
| **1,343 automated tests** | run on every commit |
| **Live contract tests against Companies House, daily** | so an upstream field-shape change surfaces as a red build, not as a customer's bad data |
| **Circuit breaker + response caching** | upstream degradation doesn't cascade to you |
| **[Public status page](https://status.registrum.co.uk)** | independently monitored, not self-reported |
| **Open-source client + examples** | MIT, no obfuscation — the [MCP server](https://github.com/vdmeu/registrum-mcp) is a thin client you can read end to end |

We are a UK-only, single-jurisdiction API. If you need 30 countries at shallow depth, we are
the wrong tool and we'd rather say so here than waste your afternoon.

---

## Try it

- **Hosted MCP, no account at all:** `https://registrum.co.uk/api/mcp`
- **Browse with no key at all:** any company page on [registrum.co.uk](https://registrum.co.uk/?utm_source=github&utm_campaign=org)
- **Free API key, no card:** [registrum.co.uk](https://registrum.co.uk/?utm_source=github&utm_campaign=org) — for when the anonymous caps get in your way
- **API reference:** [api.registrum.co.uk/docs](https://api.registrum.co.uk/docs)
- **Live plans and limits:** [`GET /v1/plans`](https://api.registrum.co.uk/v1/plans) — the
  authoritative source; we don't restate prices in docs, because they drift

Questions, bug reports, or a field we parse wrong: open an issue on any repo here, or
support@registrum.co.uk.

<sub>Companies House data is published under the
[Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/).
Registrum is an independent product and is not affiliated with or endorsed by Companies House.</sub>
