# SAP_FI-CO

A Claude Code skill that triages an SAP FI/CO support ticket before anyone logs into the system.

Given a pasted ticket, incident, or error message, it produces one short card: ranked root-cause hypotheses, an ordered read-only check sequence with T-codes and tables, and — when the ticket is too thin to work — the clarifying question to send back to the requester.

## What it is not

This does not answer tickets, and it does not read your system. It is a structuring aid for a consultant who already knows FI/CO: it makes the first twenty minutes in the system deliberate rather than exploratory, and it stops you investigating a ticket that cannot yet be investigated.

Everything it emits is a hypothesis to test. The T-codes are starting points, not answers, and they vary by release — the skill is instructed to flag ECC/S4HANA differences rather than silently picking one. It will not invent document numbers, message IDs, or SAP Note numbers.

If you do not know FI/CO, this will not make you competent in it. It will just help you lose less time.

## Install

Copy the skill directory into your project's `.claude/skills/`:

```bash
cp -r .claude/skills/sap-fi-co /path/to/your/project/.claude/skills/
```

Or into your personal skills folder, to have it available in every project:

```bash
cp -r .claude/skills/sap-fi-co ~/.claude/skills/
```

Claude Code picks up new skills without a restart. Invoke it with `/sap-fi-co`, or just paste a ticket and let Claude load it automatically — the description covers pasted error messages.

The invocation name comes from the directory name, so if you rename `sap-fi-co/`, the command changes with it.

## Example output

Illustrative only, from a deliberately thin ticket: *"User can't post an invoice, gets an error about the period. Urgent."*

```
TICKET SUMMARY
A user is blocked posting a vendor invoice and reports a period-related error.
No document, company code, or message number was supplied.

WHAT'S MISSING
Exact message class and number; company code; posting date attempted;
whether other users are affected.

MOST LIKELY CAUSE
1. Posting period closed for the account type — period/timing — "error about
   the period" most often means F5 201, raised per account type
2. MM period not rolled — period/timing — invoice posting also checks the MM
   period, which closes separately from FI
3. Authorization group on the open period — authorization — the period is
   open, but restricted to a group the user is not in

CHECK SEQUENCE
1. Ask the user for the message long text — names the account type and
   company code outright, and rules out 2 and 3 immediately
2. OB52 / T001B — the variant row for that account type and the "+" row —
   rules out 1
3. MMRV — current MM period vs the posting date — rules out 2
4. OB52 authorization-group column — rules out 3

QUESTION TO SEND BACK
Hi — to get this moving I need a bit more detail. Could you send:
the full error message (the long text, or a screenshot of the whole screen),
the company code, and the posting date you entered? Also, has this worked
for you before, and do your colleagues see the same thing?

ESCALATION
If the period is open for the right account type and the MM period is
current, stop and search the Support Portal for the message number before
opening config.
```

## Confidentiality

This repository contains no client-specific configuration, and no data from any engagement. The T-codes, tables, and message IDs in it are SAP standard; the example above is invented.

The skill itself carries a confidentiality rule: ticket content is client data, and output reused outside an engagement should be stripped of client and system names, document numbers, amounts, and vendor or customer names.

## Licence

MIT. See [LICENSE](LICENSE).
