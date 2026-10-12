# Decision 002: Sales, Payments, and Customer Balances

- **Status:** Proposed
- **Date:** 2026-10-12

## Context

Retail shops may receive full payment immediately or allow a customer to pay part of a sale total and settle the remainder later. These records must remain consistent across the dashboard, sales history, and customer balance views.

## Proposed Decision

- Store the sale total, amount paid, and outstanding balance.
- Validate that amount paid does not exceed the total unless a separately approved overpayment policy exists.
- Require a customer when a sale leaves an outstanding balance.
- Provide a query/view for customers with outstanding balances.
- Define how later payments, refunds, voids, and corrections affect balances before implementing those workflows.
- Keep sale creation, payment values, and stock deduction consistent; avoid partially saved transactions.

## Consequences

- Credit sales can be traced to a customer.
- The application can report outstanding balances without manual recalculation.
- Payment history and correction rules need careful design and automated tests.

## Alternatives Considered

- Store only a manually edited customer balance. Not recommended because it cannot reliably explain which sales created the balance.

## Approval

Pending product and engineering review.
