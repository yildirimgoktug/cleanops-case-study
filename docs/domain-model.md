# Domain model

The database has 87 concrete models across 16 apps. The diagram below is
the conceptual core, not the full schema: join tables, history tables,
labels and settings are left out.

```mermaid
erDiagram
    PROVIDER_COMPANY ||--o{ BUILDING : owns
    PROVIDER_COMPANY ||--o{ CUSTOMER : serves
    PROVIDER_COMPANY ||--o{ USER : "employs (membership / assignment)"
    CUSTOMER }o--o{ BUILDING : "operates in"
    CUSTOMER ||--o{ CUSTOMER_MEMBERSHIP : has
    USER ||--o{ CUSTOMER_MEMBERSHIP : holds
    CUSTOMER_MEMBERSHIP ||--o{ BUILDING_ACCESS : "per building: sub-role, overrides"
    BUILDING ||--o{ TICKET : at
    CUSTOMER ||--o{ TICKET : "reported for"
    TICKET ||--o{ STAFF_ASSIGNMENT : "worked by"
    CUSTOMER ||--o{ EXTRA_WORK : requests
    EXTRA_WORK ||--|{ EXTRA_WORK_LINE : "cart of"
    SERVICE ||--o{ EXTRA_WORK_LINE : priced
    SERVICE ||--o{ CUSTOMER_PRICE : "agreed per customer"
    EXTRA_WORK ||--o{ PROPOSAL : "quoted by"
    EXTRA_WORK ||--o{ TICKET : "spawns operational"
    RECURRING_JOB ||--o{ OCCURRENCE : generates
    OCCURRENCE ||--o| TICKET : spawns
    CUSTOMER ||--o{ CONTRACT : signs
    CONTRACT ||--|{ CONTRACT_REVISION : "versioned as"
    CUSTOMER ||--o{ INVOICE : billed
    INVOICE ||--|{ INVOICE_LINE : contains
    INVOICE_LINE }o--o| EXTRA_WORK : claims
    INVOICE |o--o| INVOICE : "reversed by"
    USER ||--o{ TIME_ENTRY : records
```

## Main entities

- **Provider company** — the cleaning company. The system is multi-tenant
  at this level; the test data deliberately contains a second provider.
- **Customer** — a client organisation of a provider, active in one or more
  buildings. A building can host several customers.
- **Ticket** — an issue report (complaint, request, report) at one building
  for one customer, worked by assigned staff.
- **Extra work** — work outside the regular scope, always a cart of one or
  more service lines. Approved work spawns one operational ticket per line.
- **Service and customer price** — a provider service catalogue plus
  per-customer agreed prices with validity dates. Prices are snapshotted
  onto lines so a later price change does not rewrite history.
- **Proposal** — the provider's quote for a cart that is not fully covered
  by agreed prices.
- **Recurring job and occurrence** — scheduled work that generates dated
  occurrences, each of which can spawn a ticket.
- **Contract** — recurring agreed scope (locations, lines, hours, money,
  billing rhythm). Changes are stored as revisions with an effective date,
  so a past period is billed at what was agreed then.
- **Invoice** — bills a customer's earned extra work and contract periods.
  See [invoicing.md](invoicing.md).
- **Time entry** — hours worked by an employee, independent of tickets.
- **Audit log** — field-level changes with actor, reason and a snapshot of
  the actor's scope.

## State machines

Every status change goes through a state machine that holds an explicit
table of allowed moves and the roles allowed to make each one. A move not
in the table is refused with a readable message and a stable error code.
Each accepted move writes a history row.

### Ticket

```mermaid
stateDiagram-v2
    [*] --> OPEN
    OPEN --> ACKNOWLEDGED
    OPEN --> IN_PROGRESS
    ACKNOWLEDGED --> IN_PROGRESS
    ACKNOWLEDGED --> ON_HOLD
    IN_PROGRESS --> ON_HOLD
    ON_HOLD --> IN_PROGRESS
    IN_PROGRESS --> WAITING_MANAGER_REVIEW: work done (evidence required)
    IN_PROGRESS --> WAITING_CUSTOMER_APPROVAL: work done (evidence required)
    WAITING_MANAGER_REVIEW --> WAITING_CUSTOMER_APPROVAL
    WAITING_MANAGER_REVIEW --> IN_PROGRESS
    WAITING_CUSTOMER_APPROVAL --> APPROVED
    WAITING_CUSTOMER_APPROVAL --> REJECTED
    REJECTED --> IN_PROGRESS
    APPROVED --> CLOSED: automatic on approval
    CLOSED --> REOPENED_BY_ADMIN
    REOPENED_BY_ADMIN --> IN_PROGRESS
```

- Staff can start acknowledged work and mark their own work done, routed
  to manager review or straight to the customer depending on the building.
  They cannot approve, reject or put work on hold.
- Approval and rejection belong to the customer. When provider staff
  answer on the customer's behalf, it is recorded as a workflow override
  and needs a written reason.
- Only company admins and super admins can reopen a closed ticket.

### Extra work

```mermaid
stateDiagram-v2
    [*] --> REQUESTED
    REQUESTED --> CUSTOMER_APPROVED: all lines have agreed prices (system)
    REQUESTED --> UNDER_REVIEW: needs a quote
    UNDER_REVIEW --> PRICING_PROPOSED
    PRICING_PROPOSED --> CUSTOMER_APPROVED
    PRICING_PROPOSED --> CUSTOMER_REJECTED
    PRICING_PROPOSED --> UNDER_REVIEW
    CUSTOMER_REJECTED --> UNDER_REVIEW
    CUSTOMER_APPROVED --> IN_PROGRESS: system
    IN_PROGRESS --> COMPLETED: system
    COMPLETED --> IN_PROGRESS: system reconcile
    REQUESTED --> CANCELLED
    UNDER_REVIEW --> CANCELLED
    PRICING_PROPOSED --> CANCELLED
    CUSTOMER_APPROVED --> CANCELLED
    IN_PROGRESS --> CANCELLED
```

- The route is decided once for the whole cart: if every line resolves to
  an agreed customer price, the request is approved like an order;
  if any line does not, the whole cart goes to a proposal.
- Once operational tickets exist, "is the work done?" is answered by those
  tickets only. The extra-work status follows them through system-only
  moves rather than being set by hand, so the two cannot disagree.
- Staff never see prices, proposals or commercial notes, only the
  operational tickets.

The proposal attached to an extra-work request has its own small machine:

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> SENT
    DRAFT --> CANCELLED
    SENT --> CUSTOMER_APPROVED
    SENT --> CUSTOMER_REJECTED
    SENT --> CANCELLED
```

A rejected proposal is replaced by a new draft, never moved backwards.

### Invoice

```mermaid
stateDiagram-v2
    [*] --> DRAFT: generated from earned work
    DRAFT --> ISSUED: issue
    ISSUED --> DRAFT: un-issue (only if not yet numbered)
    ISSUED --> SENT: send (number allocated, PDF frozen)
    SENT --> [*]
    note right of SENT
        Immutable. Corrected only by a reversal:
        a new, negated invoice that is born ISSUED
        with its own number.
    end note
```

Details in [invoicing.md](invoicing.md).
