# Invoicing

An invoice bills one customer (optionally one building) for work that has
been earned and not yet billed, plus contract periods that are due. The
design follows Dutch invoicing practice: invoice numbers without gaps,
sent invoices that never change, corrections only by credit note, and a
document that reads the same years later.

## Lifecycle

`DRAFT → ISSUED → SENT`, forward only, with two extra moves: un-issue
(`ISSUED → DRAFT`) and reversal of a sent invoice. See the state diagram in
[domain-model.md](domain-model.md#invoice).

- **Draft.** Created by generation, either by an operator or by the daily
  invoice run for customers whose billing day is today. Generation
  *claims* each piece of work it bills in the same transaction that
  creates the draft, so running it twice cannot bill anything twice.
  Drafts are freely editable.
- **Issued.** Content is settled, but there is still no number.
- **Sent.** The number is allocated, the PDF is frozen, and the invoice
  becomes visible in the customer portal. From here it is immutable.

Issue, send, un-issue and reverse are company-level acts (company admin or
super admin). Building managers can generate, edit and preview drafts. The
permission is checked twice on purpose: once in the view and again inside
each state-machine function.

## Gapless numbering

Numbers have the form `YYYY-NNNN`, one sequence per provider company per
year. The design choices:

1. **A dedicated counter row per (company, year).** Allocation locks that
   one row, increments it and returns the value. Concurrent sends queue on
   the lock; there is no "max + 1" query that two requests could both read.
2. **The number is born at send, not at issue.** An issued-but-unsent
   invoice has no number, so returning it to draft consumes nothing. The
   numbered set is exactly the set of committed documents.
3. **Allocation shares the transaction with the status change.** If the
   send fails, the number is rolled back with it, so no gap appears.
4. **The year is the year of allocation**, not the billing period. An
   invoice sent in January for December work takes the new year's
   sequence.
5. **A numbered invoice can only move forward.** Un-issue is refused if a
   number is present, because dropping it would leave a gap.

```text
# Illustrative pseudo-code, not the project's source
@atomic
def send(invoice, actor):
    require_company_admin(actor)
    inv = lock_row(invoice)                       # stale or concurrent status is refused
    require(inv.status == ISSUED)
    require(inv.customer.has_billing_address)
    if inv.number is None:
        seq = lock_row(sequence(inv.company, current_year()))
        seq.last += 1
        inv.number = f"{seq.year}-{seq.last:04d}"
    inv.status, inv.sent_at = SENT, now()
    save(inv)
    freeze_pdf(inv)                               # same transaction: no SENT invoice without its document
```

Contracts have their own gapless sequence (`CNT-YYYY-NNNN`) built the same
way but deliberately separate, allocated at creation and keyed on the
contract's start year.

## Credit notes by reversal

A sent invoice is never edited or deleted. A correction creates a
**reversal**:

- a new invoice with negated totals and negated mirror lines, born
  `ISSUED` with its own number from the same sequence;
- linked to the original, which stays `SENT` on the books;
- the original's claimed work is released back to the unbilled pool, so it
  can be invoiced again correctly;
- a reversal cannot itself be reversed or un-issued, and an invoice can be
  reversed only once.

The reversal's lines carry no link to the work, because it is a monetary
counter-entry only. Its PDF refers to the original invoice number instead
of repeating a specification.

## Frozen PDFs

Before freezing, the PDF was rendered from live data on each download, so
a renamed customer or a changed label could change a document that had
already been sent. The rule now is: snapshot it, store it, never
recompute.

- The PDF is rendered once, at send, inside the send transaction. If the
  file or its digest cannot be written, the send fails.
- The stored bytes are kept with their SHA-256 digest, page count and
  freeze time. The digest is an integrity witness for tests and audit and
  is not exposed through the API.
- Drafts and issued invoices keep rendering fresh, because they can still
  change.
- Invoices sent before the feature existed are frozen on first access
  under a row lock, or in bulk by an operational command. Nothing is ever
  re-frozen.

The document itself is a one-page summary plus a specification annex
grouped by building, department and work type. Page one and the annex read
from the same in-memory structure, so their counts cannot disagree, and
tests check every page of a multi-page document for fields the customer
must not see.

## One billing rule

Extra work carries a quoted amount and, for hourly work, a final amount
from actual hours. The billed ("earned") amount is the final amount when it
exists and the quoted amount otherwise. That rule is defined once and used
by every money surface: dashboards, the invoice list, and generated lines.
During an end-to-end audit, a test chain followed one job from quote to
actual hours, invoice and credit note and reconciled to the cent at every
step.
