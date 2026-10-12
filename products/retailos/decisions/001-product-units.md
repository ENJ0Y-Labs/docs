# Decision 001: Product Units and Base-Unit Inventory

- **Status:** Proposed
- **Date:** 2026-10-12

## Context

Retail shops sell the same product in different units, such as individual pieces, packs, or cartons. Treating each unit as unrelated stock can create inaccurate inventory.

## Proposed Decision

- Store stock for physical products in one defined base unit.
- Define each selling unit with a name, the number of base units it contains, and its selling price.
- Calculate the base quantity deducted as `sold quantity × base quantity per selling unit`.
- Record the selected unit, sold quantity, applied unit price, and base quantity deducted on each sale line.
- Validate available stock on the server before completing the sale.
- Existing stock should be migrated as quantities in the chosen base unit only after the current data semantics are verified.

## Consequences

- Stock checks and deductions use a consistent unit.
- Selling units can have different prices.
- Historical sale lines retain the unit and price used at the time of sale.
- Migration and API compatibility need explicit tests.

## Alternatives Considered

- Track independent stock for every selling unit. Rejected as the default because the same physical stock can become fragmented across units.

## Approval

Pending product and engineering review.
