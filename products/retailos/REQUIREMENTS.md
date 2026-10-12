# RetailOS Requirements

- **Status:** Draft baseline; confirm against the approved PRD and current implementation.

Requirements should be testable, uniquely identifiable, and linked to implementation issues and tests.

## Sales and Time

- **RET-SALE-001:** Store sale timestamps consistently in UTC.
- **RET-SALE-002:** Calculate today's totals using the Africa/Lagos local-day boundaries.
- **RET-SALE-003:** Reject or safely correct sale timestamps that are in the future.
- **RET-SALE-004:** Validate sale quantities and stock availability on the server.
- **RET-SALE-005:** Save the actual unit price applied to each sale line.

## Products and Inventory

- **RET-INV-001:** Distinguish physical products from services.
- **RET-INV-002:** Store physical stock in a defined base unit.
- **RET-INV-003:** Support selling units with a unit name, base quantity, and selling price.
- **RET-INV-004:** Deduct stock as sold quantity multiplied by the unit's base quantity.
- **RET-INV-005:** Prevent sales that would make stock negative.
- **RET-INV-006:** Record the selected unit, sold quantity, and base quantity deducted on each sale line.
- **RET-INV-007:** Calculate low-stock alerts automatically from configured thresholds.

## Customers and Payments

- **RET-PAY-001:** Store amount paid and outstanding balance for each sale.
- **RET-PAY-002:** Reject payments greater than the sale total unless an explicitly approved overpayment workflow exists.
- **RET-PAY-003:** Require a customer for sales that leave an outstanding balance.
- **RET-PAY-004:** Provide a view of customers with outstanding balances.
- **RET-CUS-001:** Define safe customer deletion behavior when sales or balances exist.

## Reporting and Client Support

- **RET-REP-001:** Support filtering or grouping sales by payment method.
- **RET-REP-002:** Return an explicit insufficient-data state when a comparison cannot be calculated.
- **RET-API-001:** Provide a health or readiness endpoint so clients can distinguish startup from availability.
- **RET-API-002:** Return product and stock information together where the client workflow requires a combined record.
- **RET-API-003:** Provide sensible defaults for frequent products, quick-add services, default units, and recent customers.

## Acceptance Criteria

Add scenario-based acceptance criteria and automated tests for each requirement before marking it complete. This list is a starting baseline, not confirmation that the current server already satisfies these requirements.
