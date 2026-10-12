# API Conventions

These conventions are proposed defaults. Confirm them against the existing RetailOS server before enforcing them.

## General

- Use predictable resource-oriented URLs and consistent HTTP methods/status codes.
- Validate all client input on the server; client-side validation is for usability, not security.
- Return consistent JSON response shapes and actionable error messages without exposing secrets or internal stack traces.
- Use pagination for potentially large collections and document filter/sort parameters.
- Represent money consistently and avoid floating-point arithmetic for financial calculations.
- Store timestamps in UTC and document local business-day interpretation explicitly.

## Compatibility

- Document breaking changes and migration requirements.
- Keep API contracts aligned with the client application.
- Add automated tests for validation, permissions, error cases, and critical business rules.

## Inventory and Sales

- Keep base-unit conversions explicit in the contract.
- The server is authoritative for price validation, stock availability, totals, balances, and stock deductions.
- Preserve the applied unit price and relevant unit quantities in saved sale lines.
