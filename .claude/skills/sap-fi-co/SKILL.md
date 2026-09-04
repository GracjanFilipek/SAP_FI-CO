---
name: sap-fi-co
description: "Triage an SAP FI/CO support ticket: rank root-cause hypotheses, produce an ordered check sequence with T-codes and tables, and draft the clarifying question back to the requester. Use when an SAP ticket, incident, or error message is pasted or described."
---

# SAP FI/CO Ticket Triage

Turn a raw support ticket into three things: ranked root-cause hypotheses, an ordered read-only check sequence, and — when the ticket is underspecified — the exact question to send back before touching the system.

The goal is not to answer the ticket. It is to make the first 20 minutes in the system count, and to stop the consultant from investigating a ticket that can't be investigated yet.

## Step 1 — Extract the facts

Pull these from the ticket text. Mark anything absent as **MISSING** — missing facts drive Step 5.

- System and environment (P / Q / D), client
- Company code, fiscal year, posting period
- Document number(s) and the object involved (vendor, customer, GL account, asset, cost object, PO)
- **Exact message: class + number + full text** (e.g. `F5 201`, `GLT2 201`, `M8 147`, `KI 235`)
- T-code or Fiori app, and what the user was doing: posting, changing, displaying, running a job
- Scope: one user / one company code / everyone
- Timing: first occurrence, did it work before, what changed (transport, support pack, period close, new master data, new user)
- Business impact and deadline (payment run tonight? close on day 3?)

**Never invent a document number, message ID, amount, or SAP Note number.** If it is not in the ticket, it is MISSING. A fabricated message ID sends someone down a wrong path for a day.

## Step 2 — Classify into a root-cause bucket

Assign one primary bucket, plus a secondary if genuinely plausible.

| Bucket | Signature |
|---|---|
| **Master data** | Fails for one vendor/customer/GL/cost object, works for others |
| **Configuration** | Fails for a whole company code, doc type, or account range; often after a transport |
| **Period / timing** | Fails for a date range or since month-end; works with a different posting date |
| **Authorization** | Fails for one user, works for another with the same input |
| **Workflow / approval** | Document exists but sits; nobody has it in an inbox; VIM/MRBR/release strategy |
| **Interface / job** | No user action involved; IDoc, tRFC, bank statement, scheduled job |
| **Custom code** | Z-program, enhancement, BADI, user exit, or a short dump in a Z object |
| **User error / expectation gap** | System behaves correctly; the user's mental model is wrong. Say so plainly and politely |
| **SAP standard defect** | Standard code, reproducible, no config explanation → Note search, then incident to SAP |
| **Not actually FI** | Root cause sits in MM, SD, HR or Basis and surfaces in FI |

The most common misclassification is treating a **user error / expectation gap** as a defect. Check that hypothesis explicitly before opening a config investigation.

## Step 3 — Symptom → first checks

Use these as starting points, not as answers. Adapt T-codes to the release (ECC vs S/4HANA) and to the client's own naming.

| Symptom | Check first |
|---|---|
| "Posting period not open" (`F5 201`) | `OB52` — the posting period variant row for that account type **and** the `+` row; `MMPV`/`MMRV` for the MM period; `T001B`. Check the authorization-group column before assuming it's closed |
| "Balancing field ... not filled" (`GLT2 201`) | Document splitting: item category assigned to the GL account, doc type → business transaction/variant, splitting rule, default profit center |
| "Account requires assignment to a CO object" (`KI 235`) | GL account's cost element category, `OKB9` default assignment, field status, whether a real CO object was passed at all |
| "Field ... is required / not allowed" | Two field statuses collide: GL master field status group (`FS00`, via `OBC4`) and posting-key field status (`OB41`). The stricter one wins |
| Tax errors (`FF`/`FS` class) | `FTXP` for the tax code in that country, tax account determination (`OB40`), whether the master data carries a default tax code |
| MIRO invoice blocked for payment | `MRBR` for the block reason; tolerance keys (`OMR6`); compare invoice vs PO price and GR quantity in `EKBE` |
| `F110` proposes nothing for a vendor | Payment method in vendor company-code data, payment block, house bank / bank determination in `FBZP`, due date vs run date, minimum amount, currency, ranking order |
| Duplicate invoice warning | Duplicate-check flag on the vendor master, and the duplicate-check configuration in MM invoice verification |
| Vendor/customer can't be used | Central and company-code posting blocks, deletion flags, purchasing/sales block, `BP` status in S/4 |
| Depreciation not posted | Depreciation run status (`AFAB`), period already posted, asset capitalized date, depreciation key, deactivation date |
| "Document number not in range" | `FBN1` for that doc type **and fiscal year** — number ranges are year-dependent and expire silently at year change |
| One user gets an authorization failure | Have **that user** run `SU53` immediately after the failure; then `SUIM` / `ST01` trace. Do not read your own `SU53` |
| Document sits in workflow, nobody sees it | `SWI1` filtered by date, agent determination, substitutes, org assignment; refresh the org buffer before concluding |
| IDoc / interface failure | `WE02`/`WE05` status and status text, `BD87` for reprocessing, partner profile in `WE20`, `SM58` for tRFC |
| Job failed overnight | `SM37` job log first, then `ST22` for dumps and `SM13` for update terminations at the same timestamp |
| Report figures "wrong" | Ledger, currency type, and selection dates before anything else. In S/4, confirm whether the two sources being compared are `ACDOCA` and a compatibility view |

**CO-specific:** cost center lock indicators and validity dates; statistical vs real postings; whether the assessment/distribution cycle has run for the period; characteristic derivation for CO-PA.

## Step 4 — Sequence the checks

Order them so each step is cheap and can eliminate a bucket:

1. **Read-only reproduction first** — display the document, read the *long text* of the message and its technical information (message class + number). The long text often names the config transaction outright.
2. **Scope test** — does it fail for another user, another company code, another document? This separates master data / authorization / config in one move.
3. **Time test** — does it work with a different posting date? Separates period issues from everything else.
4. **Then, and only then**, open configuration.

Note which step would falsify each hypothesis. A check that can't disprove anything isn't worth running.

## Step 5 — Draft the reply to the requester

If critical facts are MISSING, the deliverable is a message, not an investigation. Ask for the smallest set that unblocks work — typically:

- The full error message including its technical details, or a screenshot of the whole screen
- Document number, company code, fiscal year
- The exact steps taken, with the values entered
- Whether it worked before, and roughly when it stopped
- Whether other users see the same thing

Write it as a short, plain, non-condescending message the consultant can send as-is. No more than five questions. Do not ask for something already in the ticket — that is how support consultants lose credibility with the business.

## Step 6 — Output format

```
TICKET SUMMARY
<one or two sentences, in the business's language, not SAP's>

WHAT'S MISSING
<facts absent from the ticket, or "nothing critical">

MOST LIKELY CAUSE
1. <hypothesis> — <bucket> — <why this ticket points here>
2. <hypothesis> — <bucket> — <why>
3. <hypothesis> — <bucket> — <why>

CHECK SEQUENCE
1. <T-code / table> — <what to look at> — <what it rules out>
2. ...

QUESTION TO SEND BACK
<ready-to-send message, or "none needed">

ESCALATION
<when to stop investigating and search SAP Notes or raise to SAP / another module>
```

Keep the whole card short enough to read before opening the system. If a hypothesis needs a paragraph to justify, it is not hypothesis #1.

## Accuracy rules

- Never state an SAP Note number unless the user supplied it. Say "search the Support Portal for `<terms>`" instead.
- Flag T-codes and tables that vary by release, and say when something differs between ECC and S/4HANA rather than picking one silently.
- Distinguish what the message *says* from what you are *inferring*. A guess labelled as a guess is useful; a guess labelled as a fact costs a day.
- If the ticket is genuinely ambiguous between two buckets, say so and give the one check that separates them.

## Confidentiality

Ticket content is client data. When output from this skill is reused for anything outside the engagement — notes, a colleague, a blog post, a knowledge base — strip client and system names, document numbers, amounts, and vendor/customer names, and keep only the generic pattern. Ask before assuming a reuse is in scope.
