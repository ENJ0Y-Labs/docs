# RetailOS User Flows

These flows are initial drafts. Validate them with shopkeepers and update them when behavior is approved.

## 1. Record a Sale

1. The cashier opens checkout.
2. The cashier searches for a product or selects a frequent item/quick-add service.
3. The cashier selects the selling unit and enters a quantity.
4. The application shows the unit price and line total.
5. For physical products, the server validates available stock using the base-unit conversion.
6. The cashier selects or enters a customer when required.
7. The cashier enters the amount paid and payment method.
8. The server validates the sale and records the sale, lines, payment details, and stock movement atomically.
9. The application displays the saved result and updated totals.

## 2. Record a Sale With a Balance

1. The cashier creates a sale and enters an amount paid below the total.
2. The application calculates the outstanding balance.
3. The cashier selects a customer; the application must not save a credit sale without one.
4. The server validates and saves the sale and balance.
5. The customer's outstanding balance reflects the transaction.

## 3. Review Stock

1. The shopkeeper opens the product/stock list.
2. The application displays stock in the product's base unit and any supported selling units.
3. Low-stock status is derived from the current quantity and configured threshold.
4. A sale reduces physical stock by the converted base quantity; service sales do not affect physical stock.

## 4. Review Today's Sales

1. The shopkeeper opens the dashboard.
2. The server determines the current Africa/Lagos business-day range.
3. The dashboard total uses that same range as the sales list/report.
4. The application clearly indicates when a comparison has insufficient historical data.

## 5. Find a Customer With Outstanding Balance

1. The shopkeeper opens the outstanding balances view.
2. The application lists customers with outstanding balances.
3. The shopkeeper opens a customer to review relevant sales and balance details.

## Error and Recovery Cases

- Insufficient stock: reject the sale with a clear message and do not partially record it.
- Invalid quantity or price: reject invalid values and preserve existing records.
- Payment greater than total: reject unless an approved workflow explicitly permits it.
- Missing customer for a credit sale: explain that a customer is required.
- Server starting or unavailable: show a clear status and avoid duplicate sale submissions.
