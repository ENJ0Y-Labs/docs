# RetailOS Product Requirements Document (PRD)

- **Status:** Draft
- **Owner:** Trexave product team
- **Last updated:** 2026-10-12

## 1. Summary

RetailOS is intended to help small retail businesses manage sales, products, stock, customers, and day-to-day store records in one application.

This document is a starting draft. Validate the details with intended shopkeepers before treating assumptions as approved requirements.

## 2. Problem

Small shops need a practical way to record sales and stock movements, track customer debts, and understand daily performance without relying on disconnected notebooks or manual calculations.

## 3. Target Users

- **Shop owner:** manages the store, reviews performance, and follows up on outstanding balances.
- **Shop attendant/cashier:** records sales quickly and accurately.

## 4. Goals

- Make common sales workflows fast and understandable.
- Keep stock quantities and stock deductions accurate.
- Record full and partial payments and outstanding customer balances.
- Provide useful daily sales and business summaries.
- Reduce duplicate entry and avoid silent data inconsistencies.

## 5. Out of Scope for the Initial Release

Confirm scope with the product owner before committing to these exclusions. Potentially deferred areas include multi-branch enterprise management, payroll, and full accounting integrations.

## 6. Core Workflows

1. Sign in and access the store profile.
2. Create and manage products, services, and stock.
3. Find or select products and services during checkout.
4. Record quantities, prices, payment method, amount paid, and customer where applicable.
5. Validate stock and record the sale consistently.
6. Review sales history, stock alerts, and customer balances.
7. Review daily and period-based summaries.

See [User Flows](USER-FLOWS.md) for workflow details.

## 7. Product Behavior and Rules

- Physical products affect inventory; services do not deduct physical stock.
- Stock is maintained in a defined base unit. Selling a larger unit must deduct the corresponding number of base units.
- Each completed sale line records the selected unit, quantity, applied unit price, and base quantity deducted where applicable.
- A sale's amount paid must not exceed its total.
- A sale with an outstanding balance requires a customer.
- Today's reporting uses the store's local business day, with timestamps stored consistently in UTC and interpreted using the Africa/Lagos day boundaries where applicable.
- Important inventory and payment operations must be validated server-side.

These rules are initial product direction and must be reconciled with the approved requirements and implementation decisions before release.

## 8. Success Measures

Define baselines and targets before launch. Candidate measures:

- Time required to record a typical sale.
- Percentage of sales recorded without correction.
- Difference between recorded stock and physical stock counts.
- Percentage of sales with complete payment and customer records where required.
- Weekly active stores and repeat usage.

## 9. Constraints and Assumptions

- The initial target market is small retail businesses in Nigeria.
- The client and server are separate codebases but must implement the same product behavior.
- Network reliability, onboarding effort, device constraints, and data protection need validation with users.

## 10. Risks

- Incorrect unit conversion can cause inventory discrepancies.
- Inconsistent date boundaries can make dashboard totals disagree with sales lists.
- Ambiguous debt and customer-deletion rules can damage trust in records.
- Requirements drifting between client and server can create incomplete workflows.

## 11. Open Questions

- Which shop types and workflows are the launch priority?
- What devices and connectivity conditions must the first release support?
- Which roles and permissions are required for launch?
- What is the approved policy for refunds, voids, and corrections?
- What are the exact acceptance targets for performance and reliability?

## 12. Approval

- Product owner: Pending
- Engineering review: Pending
- User validation: Pending
