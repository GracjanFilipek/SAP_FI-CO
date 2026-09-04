# SAP_FI-CO

A Claude Code skill that triages an SAP FI/CO support ticket before anyone logs into the system.

Paste a ticket, incident, or error message. You get back one short card: ranked root-cause hypotheses, an ordered read-only check sequence with T-codes and tables, and the clarifying question to send back when the ticket is too thin to work.

## Scope

The skill structures your thinking before you log in. It never touches your system, and it will not answer the ticket for you. If you already know FI/CO, it makes your first twenty minutes count and stops you investigating a ticket that nobody can investigate yet.

Treat every line it emits as a hypothesis to test. T-codes are starting points and they vary by release, so the skill flags ECC/S4HANA differences instead of picking one for you. It invents no document numbers, message IDs, or SAP Note numbers.

If you do not know FI/CO, this will not make you competent in it. It will save you time you would otherwise lose.

## Install

Copy the skill directory into your project's `.claude/skills/`:

```bash
cp -r .claude/skills/sap-fi-co /path/to/your/project/.claude/skills/
```

Or into your personal skills folder, to have it available in every project:

```bash
cp -r .claude/skills/sap-fi-co ~/.claude/skills/
```

Invoke it with `/sap-fi-co`, or paste a ticket and let Claude load the skill on its own, since the description covers pasted error messages.

Claude Code watches skill directories and picks up edits live. If you had to create `~/.claude/skills/` for the copy above, restart Claude Code once so it starts watching the new directory.

The invocation name comes from the directory name, so if you rename `sap-fi-co/`, the command changes with it.

## Example output

Illustrative only. The ticket below is thin on purpose: *"User can't post an invoice, gets an error about the period. Urgent."*

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

This repository contains no client-specific configuration, and no data from any engagement. The T-codes, tables, and message IDs in it are SAP standard, and nothing in the example above comes from a real ticket.

The skill itself carries a confidentiality rule: ticket content is client data. Before you reuse output outside an engagement, strip client and system names, document numbers, amounts, and vendor or customer names.

## Licence

MIT. See [LICENSE](LICENSE).
