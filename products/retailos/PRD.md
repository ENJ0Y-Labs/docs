# TX RetailOS — Product Requirements Document (PRD)

**Document status:** Draft for product and engineering alignment  
**Product:** TX RetailOS  
**Product owner:** Trexave Technologies  
**Last reviewed:** 12 October 2026  
**Repositories reviewed:** [`retailos-client`](https://github.com/trexave-labs2/retailos-client) and [`retailos-server`](https://github.com/trexave-labs2/retailos-server)  
**Review basis:** Public `main` branches as observed on 12 October 2026. Client revision: `72622ecf935c92f7c0a2cbfc0302c8dc5748e40b`. Server revision: `5221970eaeda73b9e9fc4e3cd60d5d6115dfe695`. This document is a product/engineering specification, not a claim that every requirement is already implemented or production-ready.

---

## 1. Executive summary

TX RetailOS is a retail operations and point-of-sale (POS) system for small shops. Its first job is to let a shop owner and staff reliably record sales, maintain stock, track customers and unpaid balances, spot low-stock products, and understand the day's activity without doing the same arithmetic repeatedly in a notebook, calculator, and spreadsheet.

The product should optimize for reliable records and quick everyday operations before adding advanced intelligence. A trustworthy sale must save the correct amount, payment, customer balance, selling unit, and stock deduction together. A failed or retried request must not create duplicate sales or silently corrupt inventory. Dashboard figures must be derived from the same persisted records as sales history.

### Immediate architecture finding

The two repositories currently contain **two independently implemented backends**:

- The Next.js client repository contains its own PostgreSQL schema, connection pool, authentication, and `/api/*` route handlers that directly query and mutate PostgreSQL.
- The Flask server repository separately contains application routes, services, SQLAlchemy models, migrations, and tests for core retail operations.

They are not interchangeable implementations of one API. They differ in ID types, authentication method, product-type values, units, payment fields, store modeling, and available endpoints. For example, the client uses UUIDs, a username/password JWT cookie, `PRODUCT`/`SERVICE`, and a single `store_settings` row; the Flask server uses integer model IDs, email/password Flask sessions, `Physical`/`Service`, and store-scoped records. The client also exposes purchases, user administration, notifications, and reports that the Flask server does not currently expose as equivalent modules.

**Release-blocking decision:** choose one source of truth before wiring the applications together. This PRD recommends the Flask server as the authoritative business API and database owner, with the Next.js application acting as the user interface and, if needed, a thin same-origin proxy. This recommendation is based on the server's existing service layer, transaction handling, store authorization, migrations, and test coverage. It is a proposed target, not a description of today's integrated behavior.

Do not point the UI at the Flask server and assume the integration is complete. A contract adapter, role support, missing API capabilities, and a deliberate data/schema migration are required.

## 2. Product vision

### 2.1 Mission

Give small retailers a clear, dependable view of what they have in stock, what they have sold, what customers owe, and what requires attention today.

### 2.2 Problem

Small shops often rely on manual or disconnected records. That can lead to inaccurate stock, repeated calculations, unclear customer debts, hard-to-trace stock changes, and no reliable daily overview.

### 2.3 Product promise

RetailOS should answer these questions quickly and accurately:

1. What stock do I have now?
2. What did the shop sell today?
3. Which payment methods brought in the money?
4. Which customers still owe money, and how much?
5. Which products need restocking?
6. What happened, who recorded it, and when?

### 2.4 Product principles

- **Correctness before cleverness:** business totals and stock deductions are calculated and persisted by the authoritative backend.
- **Fast shop-floor workflows:** common actions must take few, clear steps and work on a normal laptop-sized screen and narrower mobile screens.
- **One record, one meaning:** the same sale should produce the same total, payment balance, stock effect, receipt, and report wherever it appears.
- **Traceable changes:** stock adjustments, sale cancellation, payment collection, and purchase approval must be attributable and auditable.
- **Useful error messages:** tell the user what failed and how to recover without exposing internal exceptions.
- **Incremental delivery:** ship reliable authentication, inventory, sales, alerts, and the Daily Business Brief before expanding into heavier reporting and settings.

## 3. Goals and non-goals

### 3.1 Goals for the first usable release

1. Authenticate users and enforce ADMIN, MANAGER, and EMPLOYEE permissions on the server.
2. Maintain a store-scoped product catalog with physical products, services, stock, base units, and selling units with independent prices.
3. Record a multi-line sale reliably, including selected unit, quantity, sale-time price, payment method, amount paid, and outstanding balance.
4. Prevent overselling and duplicate sale processing; record the stock movement caused by each physical sale.
5. Associate unpaid sales with a customer and maintain a reliable debt and repayment history.
6. Support manual stock adjustments and automatic low-stock alert synchronization.
7. Provide a dashboard and Daily Business Brief based on persisted data and Africa/Lagos calendar-day boundaries.
8. Provide sale history and a structured, printable-text receipt.
9. Make schema changes repeatable through migrations and prove critical workflows with automated tests.
10. Deploy with production secrets, HTTPS, database backups, health checks, and an operational rollback plan.

### 3.2 First-release non-goals

The first release does not need to include machine-learning forecasts, automated purchasing recommendations beyond simple restock signals, a payment gateway, subscription billing, full accounting, expenses management, multi-store switching for a single user, offline synchronization, device management, or PDF/printer-specific receipt integrations. These may be evaluated later from real usage.

### 3.3 Planned follow-on capabilities

Purchases and supplier records, detailed daily/monthly/yearly reports, notification preferences, data export, account profile management, device/session management, and broader business insights may be delivered after the operational core is stable. Some already appear in the Next.js UI, but their presence in the UI does not prove that they are integrated with the Flask server or ready for release.

## 4. Users and permissions

RetailOS V1 has three application roles, already represented in the client UI.

| Role | Intended responsibility | Permissions |
|---|---|---|
| **ADMIN** | Owner or trusted administrator | All store operations; create and manage users; assign roles; activate/deactivate users; manage store profile; view reports and audit activity. |
| **MANAGER** | Day-to-day shop supervisor | Manage products and stock; record sales; manage customers and debt collection; create/approve purchases when purchases are enabled; view reports and operational activity; update permitted store settings. Cannot manage ADMIN users or grant ADMIN privileges. |
| **EMPLOYEE** | Cashier/shop assistant | Browse active products and services; record sales; select customers; record repayments if authorized by the store; view permitted sales history and operational alerts. Cannot manage users, approve purchases, or access manager-only reports. |

### Permission rules

- Authorization must be enforced in the backend for every protected operation. Hiding a link in the UI is not authorization.
- The logged-in user must be assigned to the store whose data is being accessed.
- The current client uses a single store-settings row; the target data model must instead consistently associate business records with a store. V1 can limit each user to one assigned store, but every store-owned record still needs a `store_id` boundary.
- Users must not be able to promote themselves, disable the last active ADMIN, or modify users outside their store.
- Deactivation must block new sessions and invalidate/reject existing sessions within a defined short window. A seven-day signed token that keeps stale role/active claims without revalidation is not sufficient for immediate staff removal.

## 5. Scope and release priority

### P0 — Required for a reliable first release

- Source-of-truth architecture and schema migration plan.
- Authentication, current-user/session endpoint, logout, role enforcement, store authorization.
- Product/service catalog, stock quantity, base unit, low-stock threshold, selling units and prices.
- POS cart, quantity validation, physical stock checks, service price override, multi-line checkout.
- Payment method, amount paid, balance, customer-required-for-debt rule, payment/repayment ledger.
- Sale history, receipt data/printable text, safe cancellation rules.
- Inventory movements for opening stock, manual adjustments, sales, and reversals.
- Low-stock alert creation/deduplication/resolution based on each product's threshold.
- Dashboard and Daily Business Brief using correct Lagos-day boundaries.
- Repeatable database migrations, automated critical-path tests, production environment validation, backup and health-check procedure.

### P1 — Follow-on after P0 is stable

- Purchase entry and approval, with stock and cost effects applied once.
- Detailed reports with payment breakdown, top items, employee performance, profit, and selectable periods.
- Notifications generated from defined business events, with read state.
- Expanded customer history/search, store profile, audit-log UI, and export.
- Improved quick actions and sales defaults: frequent products, service shortcuts, default units, and recent customers.

### P2 — Later evaluation

- PDF and printer-specific receipts, payment-provider integration, expenses/accounting, advanced predictive recommendations, multiple-store membership/switching, offline-first mode with sync, device management, and notification preferences.

**Prioritization note:** The accompanying project roadmap emphasizes authentication, core entities, sales, alerts, and the Daily Business Brief as the first release spine. Reports and broader operations should not delay that release if the core workflows are not yet trustworthy.

## 6. Main user journeys

### 6.1 First-time setup and sign-in

1. An authorized deployment operator configures secrets and database access.
2. The first ADMIN account is created through a guarded, one-time setup process or another explicitly chosen server-side provisioning flow.
3. The user signs in with the agreed canonical credential format.
4. The server creates a secure session and returns the current user, role, and assigned store context.
5. The client loads the role-appropriate dashboard and protects application screens against unauthenticated access.
6. Sign-out clears the session. A deactivated account cannot continue to call APIs with an old session.

The current client uses a guarded `/api/setup` endpoint and username/password login. The Flask server currently exposes registration and email/password login. The target must select one setup and credential contract; the two implementations must not remain as competing authentication systems.

### 6.2 Start-of-day review

1. User opens the dashboard.
2. Dashboard displays today's completed sales, transaction count, active product count, stock-unit count, low-stock products, recent transactions, and role-appropriate recent activity.
3. User opens the Daily Business Brief.
4. Brief compares sales with the previous Lagos calendar day, indicates whether the comparison is available, shows open low-stock alerts, and suggests products needing attention.
5. A missing comparison baseline is shown as unavailable/null, not fabricated as `0.00%`.

### 6.3 Product setup and unit management

1. ADMIN or MANAGER creates a physical product or service.
2. They provide a name, product type, base unit, base-unit selling price, cost information where relevant, opening stock for physical goods, and low-stock threshold.
3. They configure one or more selling units, each with a unit name, quantity in base units, and independent selling price. Example: `piece = 1 base unit at ₦50`; `carton = 40 base units at ₦1,800`.
4. The server validates the units and stores all physical stock in the product's base unit.
5. Existing products remain sellable after migration, with a default unit that represents one existing base unit at the prior price.
6. Service products do not require stock and must not be blocked because their ignored stock quantity is zero.

### 6.4 POS sale

1. Cashier searches or chooses a product/service.
2. Cashier selects the selling unit and quantity. The cart displays the unit price, line total, and physical base quantity that will be deducted.
3. Cashier optionally attaches a customer for a fully paid sale. If the sale will have an outstanding balance, a customer is mandatory.
4. Cashier selects Cash, Transfer, or POS and enters the amount received. Omitted amount paid means full payment only if the client explicitly uses the agreed default contract; the server remains responsible for validating it.
5. Client sends a unique transaction/idempotency key for this checkout attempt.
6. Server validates every line, resolves the current unit and catalog price, calculates totals using decimal-safe currency arithmetic, locks/rechecks physical stock, validates payment and customer, and commits the sale, sale lines, payments, stock deductions, stock movements, and audit record in one transaction.
7. Client displays a success confirmation and receipt details, clears the cart, and refreshes product/stock data.
8. A retry with the same idempotency key returns the existing result instead of creating another sale or deducting stock a second time.

### 6.5 Debt collection / repayment

1. Staff opens “Who Owes Me” and sees only customers/sales with a positive outstanding balance.
2. Staff opens the relevant sale, sees original total, payments to date, and current balance.
3. Staff enters a positive repayment amount and payment method.
4. Server rejects zero, negative, malformed, or over-balance payments.
5. Server appends an immutable payment entry, updates the sale's paid/balance values and customer outstanding total consistently, and writes an audit record in the same transaction.
6. Fully settled debts disappear from the outstanding list but remain in customer and sale history.

### 6.6 Stock adjustment

1. ADMIN or MANAGER opens a product and chooses to add or remove stock.
2. They enter a base-unit quantity change and a reason.
3. Server validates the product/store and rejects a result below zero.
4. In one transaction, server updates stock and records previous quantity, change, new quantity, actor, reason, and timestamp.
5. Low-stock state and open alert are synchronized automatically.

### 6.7 Sale cancellation

1. An authorized MANAGER or ADMIN opens a completed sale and starts cancellation.
2. The UI requires a reason and describes the consequences.
3. The server checks current status and payment/refund state, then performs a one-time reversal. It must not restore stock twice, erase the original record, or leave customer balances incorrect.
4. If money has been collected, cancellation must be blocked until an explicit refund/reversal capability is implemented and the refund is recorded, or the product must use a separate audited refund workflow. Do not silently mark a paid sale cancelled while leaving collected money unaccounted for.
5. The cancelled sale remains visible in history with the actor, timestamp, reason, and reversal details.

## 7. Functional requirements

Requirement IDs are stable references for implementation issues and acceptance tests.

### 7.1 Authentication, session, and store context

**AUTH-01** The system shall provide a single canonical login contract and credential format for the client and API.  
**AUTH-02** Passwords shall be stored as strong password hashes and shall never be returned to the client.  
**AUTH-03** Protected API requests shall require a valid session.  
**AUTH-04** The API shall provide a current-user endpoint returning user ID, username/display name, role, and assigned store context.  
**AUTH-05** Logout shall invalidate the session.  
**AUTH-06** Deactivated users and users whose role/store assignment has changed shall not retain stale privileged access.  
**AUTH-07** First-admin setup shall be disabled after successful initialization and shall not be reopenable by an ordinary user.  
**AUTH-08** All store-owned reads and writes shall verify store access on the server. User-supplied `store_id` alone shall never establish authorization.  
**AUTH-09** The UI shall route users to role-appropriate dashboards, but direct URL access must also be checked server-side.  
**AUTH-10** API errors shall distinguish validation, unauthenticated, forbidden, not-found, conflict, and internal errors with stable codes.

### 7.2 Products, services, stock, and units

**INV-01** Each catalog item shall be either a physical product or a service. The API's canonical enum must be the same in all clients and services.  
**INV-02** Every physical product shall define a base unit, and `stock_quantity` shall always represent quantities in that base unit.  
**INV-03** Every selling unit shall define `name`, `base_quantity`, and `price`; each unit has its own price and need not be a simple multiplication of base-unit price.  
**INV-04** Existing products shall migrate to a base unit and a default one-base-unit selling unit without changing current stock value or historical sale values.  
**INV-05** Product creation/editing shall validate names, item type, non-negative prices/costs, stock quantity, threshold, unique unit names, positive unit conversion, and valid unit prices.  
**INV-06** Physical stock shall never become negative. All stock-changing operations shall be transactionally safe under concurrent requests.  
**INV-07** Service sales shall not check, reserve, deduct, or create stock movements for service inventory.  
**INV-08** Updating a product's catalog price or unit conversion shall not rewrite historical sale-line name, unit, conversion, price, or total.  
**INV-09** Products referenced by transaction history shall be archived/deactivated or protected from destructive deletion. The user must not lose records because a catalog item was removed.  
**INV-10** Low-stock threshold shall be expressed in the same base unit as stock. `stock_quantity <= low_stock_threshold` defines low stock.  
**INV-11** V1 quantities and conversion values shall use whole-number base quantities to match the current Flask model and prevent fractional stock deductions. Fractional measured quantities require an explicit future decision and consistent migration of the schema, validators, UI, and reports.  
**INV-12** The product list response shall include every field the UI needs, including item type, base unit, stock quantity, threshold, active state, current units/prices, and ID; service items must not be misclassified because a response omitted `item_type`. PATCH semantics must be consistent: either support genuine partial updates without silently resetting omitted values, or expose a clearly documented full-replacement operation.

### 7.3 Sales and POS

**SALE-01** A sale shall contain one or more line items.  
**SALE-02** Each line shall record product ID, selected unit ID/name snapshot, sold quantity, base quantity per selected unit at sale time, total base quantity deducted, actual unit price, and line total.  
**SALE-03** Server-side calculations shall determine line totals and sale total; the client shall not be trusted to submit the final total.  
**SALE-04** Currency math shall use exact decimal values or integer minor units, with a two-decimal NGN amount format. Floating-point rounding must not change persisted totals.  
**SALE-05** For physical products, required stock is `sold quantity × base quantity per selling unit`. The server shall reject the entire sale if any line lacks available stock.  
**SALE-06** The server shall use transaction/row-locking or equivalent concurrency protection so two simultaneous checkouts cannot both sell the same remaining stock.  
**SALE-07** A service may accept a checkout-specific unit-price override. The override shall be validated and saved on that sale line only. Physical product price overrides shall be rejected unless a distinct approved discount feature is specified.  
**SALE-08** Each sale shall store total, amount paid, outstanding balance, payment method, customer when applicable, creator, timestamp, status, and an immutable transaction reference/receipt number.  
**SALE-09** `amount_paid` must be greater than or equal to zero and must never exceed the server-calculated total. `balance = total_amount - amount_paid` must never be negative.  
**SALE-10** Any sale whose balance is above zero shall require a valid customer belonging to the same store.  
**SALE-11** Each checkout request shall include an idempotency key/client transaction ID unique within the store. Replays shall return the existing sale result without repeating financial or inventory effects.  
**SALE-12** Creating a sale shall atomically persist its header, lines, initial payment entry (when amount paid is positive), stock deductions, inventory movements, customer balance effect, alert synchronization, and audit event. Failure shall roll back all effects.  
**SALE-13** Valid payment methods for V1 shall be Cash, Transfer, and POS. One consistent canonical spelling/case shall be used in the API and stored data; display casing may be formatted by the UI.  
**SALE-14** The product list/cart shall support multiple lines and independent quantities/units for the same product where useful. Every quantity must be a positive whole number in V1.  
**SALE-15** After a successful sale, the client shall show a clear confirmation and receipt data, clear or reset the completed cart, and refresh affected product stock and dashboard information.  
**SALE-16** Sales history shall filter/search by receipt reference, status, payment method, date range, and customer/staff when those fields are implemented. Results shall be paginated or otherwise bounded.  
**SALE-17** Cancelling a sale shall be an audited state transition, never a delete. Restocking must happen exactly once, and customer balances must be reversed exactly once. Paid sales require a recorded refund/reversal workflow or shall be rejected from cancellation.  
**SALE-18** Receipt output shall be generated from persisted sale and sale-line data, include total/paid/balance, payment method, date/time, customer if present, units/quantities, and store identity, and provide printable text. PDF generation is out of P0 scope.

### 7.4 Customers, balances, and payments

**CUS-01** A customer shall belong to one store and have a name plus optional phone/contact and email.  
**CUS-02** Authorized users shall be able to create, list/search, view, update, and safely archive customers. Search shall match the supported name/contact fields case-insensitively.  
**CUS-03** Customer purchase history shall show sale reference, date, items, sale total, amount paid, balance, and status.  
**CUS-04** Customer outstanding total shall equal the sum of valid outstanding balances for completed, non-reversed sales. The system shall define one authoritative calculation and keep any stored aggregate synchronized transactionally.  
**CUS-05** “Who Owes Me” shall list only customers with positive outstanding debt, sorted by amount due (descending) then name, and display the store's total outstanding balance.  
**CUS-06** A payment/repayment shall append a ledger entry with sale, amount, payment method, actor, and timestamp. The ledger entry shall not be edited or deleted through ordinary UI workflows. Corrections must use explicit reversal entries.  
**CUS-07** Repayment amount must be positive, valid to two decimal places, and no greater than the outstanding balance at the time the transaction is committed.  
**CUS-08** The sale's paid amount, sale balance, customer outstanding balance, payment ledger, and audit event shall remain consistent after every repayment.  
**CUS-09** Customer deletion/archive shall be prevented while an outstanding balance remains. Records with historical sales shall be archived or retained rather than destructively deleted.  
**CUS-10** The server shall support the customer fields and history the UI requires. Client-only email/customer attributes must not disappear in the canonical API without a declared migration or UI change.

### 7.5 Inventory movements and purchases

**STOCK-01** Each inventory movement shall record store, product, movement type, quantity change, previous quantity, resulting quantity, actor, reason/reference, and timestamp. Quantity values shall be in the product's base unit.  
**STOCK-02** At minimum, movements shall be recorded for opening stock, manual adjustments, completed physical-product sales, purchase approvals, and sale reversals.  
**STOCK-03** Manual stock changes shall require a reason, reject a negative resulting balance, and execute as one transaction with the movement record.  
**STOCK-04** Stock displayed by the client shall refresh after sales, adjustments, purchase approval, or cancellation.  
**STOCK-05** If purchase management is enabled, a purchase shall contain supplier, items, quantity in an agreed base/selling unit, unit cost, total, creator, state, and timestamps.  
**STOCK-06** Purchase creation shall not add stock until authorized approval. Approval shall add stock and record movements exactly once. Repeated approval requests must be idempotent.  
**STOCK-07** Cancelling a pending purchase shall not alter stock. Cancelling or reversing an approved purchase requires an explicit reversal operation and audit trail.  
**STOCK-08** Purchases shall be limited to physical products in V1 unless service expenses are deliberately defined as a separate feature.  
**STOCK-09** A purchase approval shall validate that all referenced products belong to the same store and that quantities/costs are valid before applying any change.  
**STOCK-10** The updated product cost rule must be defined. V1 may store the last approved unit cost as a simple baseline, but the report must disclose that this is not weighted-average/FIFO cost accounting.

### 7.6 Low-stock alerts and notifications

**ALERT-01** A product is low-stock when its physical stock quantity is less than or equal to its configured threshold. Service items are excluded.  
**ALERT-02** Low-stock state shall synchronize automatically after product edits, sales, adjustments, and purchase approvals. A user shall not have to remember a manual “generate alerts” step for the dashboard to be correct.  
**ALERT-03** Only one unresolved low-stock alert per store/product/type should exist at a time. Repeated checks must not duplicate it.  
**ALERT-04** The alert shall be resolved automatically when stock rises above the threshold, or manually by a permitted user if the UI supports manual resolution. A resolved alert may recur as a new alert if the product becomes low-stock again.  
**ALERT-05** Alert lists shall be store-scoped and include ID, type, message, product reference where applicable, created time, and resolution state.  
**ALERT-06** Notifications are distinct from operational alerts. If notifications are in P1, define and implement the events that create them, the recipient rules, read/mark-all-read behavior, and audit semantics. Do not ship a notification screen backed only by a table that is never populated.

### 7.7 Dashboard and Daily Business Brief

**DASH-01** The main dashboard shall show today's completed sales total, today's transaction count, active product count, current stock units, and low-stock products. Role-specific activity may be shown only to authorized roles.  
**DASH-02** The dashboard and Daily Business Brief shall use the `Africa/Lagos` calendar day for “today,” with sale timestamps stored consistently in UTC. The day range shall be a half-open interval from local midnight to the next local midnight converted to UTC.  
**DASH-03** Future-dated sales must be rejected or corrected by the backend; they must not distort daily or period reports.  
**DASH-04** The Daily Business Brief shall compare today's sales with yesterday's total and return an explicit comparison-available indicator or null percentage when a meaningful baseline does not exist.  
**DASH-05** The brief shall report today's sales state, unresolved operational alerts, restock candidates, and a highest-performing product where there is sufficient data. Unsupported recommendations shall be omitted or marked unavailable rather than invented.  
**DASH-06** Dashboard metrics, sales history, receipts, and reports shall agree for the same date/status filters and define whether totals represent completed sales, collected payments, or outstanding balance.  
**DASH-07** Loading, empty, stale, server-waking, and error states shall be visually distinct. An API failure must not quietly look like a genuine ₦0 sales day.  
**DASH-08** Stock quantities from unlike base units shall not be combined into a single misleading total. Either show quantities grouped by base unit or omit the aggregate and use comparable metrics such as product count and low-stock count.

### 7.8 Reporting (P1)

**REPORT-01** Reports shall support daily, monthly, and yearly periods with explicit period start and end boundaries in `Africa/Lagos`.  
**REPORT-02** The API must use both lower and upper bounds for a period; a monthly report must not include later months, and a yearly report must not quietly use a different period than the UI label.  
**REPORT-03** Reports may include total sales, estimated gross profit, transaction count, payment breakdown, top items, employee performance, and a rolling seven-day trend. Every metric shall state its meaning.  
**REPORT-04** Profit shall use persisted sale-time cost basis. If cost data is missing or only the last purchase cost is available, indicate that profit is an estimate and document the limitation. Services need a defined cost treatment.  
**REPORT-05** Payment-method analysis for installments should sum actual payment ledger entries by payment method. Grouping only by the initial sale header method can misrepresent later repayments made by another method.  
**REPORT-06** Employee names and financial reporting are restricted to ADMIN and MANAGER. Employees must not gain access by opening a report URL directly.  
**REPORT-07** Period filters and chart labels shall match the returned dataset. Yearly/monthly empty states must not show placeholder claims inconsistent with actual calculations.

### 7.9 User administration, store profile, and audit

**USER-01** ADMIN can create users with username, display name, password, and an allowed role.  
**USER-02** ADMIN can update permitted profile/role/status fields and activate/deactivate users. The API shall validate role values and protect the last active ADMIN.  
**USER-03** Role changes shall take effect promptly and be recorded in audit logs.  
**USER-04** Store profile shall expose the authorized store name. Only permitted roles may update it.  
**AUDIT-01** Audit records shall capture actor, action, entity, entity ID, timestamp, and relevant non-secret details for important mutations, including user management, stock adjustment, sale creation/cancellation, payment collection/reversal, and purchase approval/reversal.  
**AUDIT-02** Audit records shall not contain passwords, session cookies, access tokens, or other secrets.  
**AUDIT-03** An ordinary application user shall not be able to edit or delete audit events.

### 7.10 Health, errors, and operational controls

**OPS-01** The service shall expose a small unauthenticated health/readiness endpoint suitable for deployment monitoring. Readiness should distinguish the process being alive from the database being usable if both checks are supported.  
**OPS-02** Protected endpoint failures shall return stable response shapes and status codes. Unexpected errors shall be logged server-side and returned as safe generic messages.  
**OPS-03** Production shall require explicit secrets and database configuration and must refuse to start when critical configuration is missing.  
**OPS-04** Cross-origin access, if direct browser-to-Flask requests are used, shall use an explicit allowlist and credential-safe cookie settings. Wildcard CORS must not be combined with credentialed requests.  
**OPS-05** Schema migrations shall run in a controlled deployment step and be verifiable before new application code depends on them.  
**OPS-06** A backup and restore runbook shall exist before real shop data is accepted.  
**OPS-07** Critical checkout paths shall be observable with request/error logging and without logging credentials or sensitive payment/customer data unnecessarily.

## 8. Business rules and calculation definitions

| Concept | Required definition |
|---|---|
| Base stock | The smallest inventory unit for a physical product. `stock_quantity` is always expressed in this unit. |
| Selling unit | A customer-facing unit with a name, a base-unit conversion quantity, and its own selling price. |
| Base quantity deducted | `sale_line.quantity × sale_line.unit_base_quantity`, only for physical products. |
| Line total | `sale_line.quantity × sale_line.price_at_sale`, rounded/represented safely to two decimal places. |
| Sale total | Sum of persisted line totals. The client does not supply the authoritative total. |
| Amount paid | Sum of valid, non-reversed payments recorded against the sale, or a transactionally maintained value reconciled against that ledger. |
| Balance | `max(total_amount − amount_paid, 0)` for an unreversed completed sale. A sale cannot accept a payment greater than its balance. |
| Customer outstanding | Sum of positive balances across the customer's completed, non-reversed sales. Any denormalized total must reconcile with sale-level balances. |
| Low stock | A physical product is low-stock when `stock_quantity <= low_stock_threshold`, where both quantities use the same base unit. |
| Today | The `Africa/Lagos` local calendar day converted to UTC bounds for timestamp comparison. |
| Daily sales | Sum of completed, non-reversed sale totals created within the current Lagos day. Do not confuse sales value with cash collected. |
| Gross profit (P1 reports) | Sale revenue minus a defined persisted cost basis. If cost basis is incomplete, label profit an estimate. |
| Cancellation | An audited reversal, not deletion. Stock, customer balance, payment/refund records, and reports must be reversed consistently and only once. |

## 9. Canonical data model — target

The exact SQL types and ID migration strategy must be settled before cutover. The business entities and relationships below are the target regardless of whether API IDs are integers or UUIDs.

| Entity | Main fields / purpose | Integrity requirements |
|---|---|---|
| User | ID, username, display name, email if selected, password hash, role/status, timestamps | Unique login identifier, hashed password, valid role, deactivated-user access blocked. |
| Store | ID, name, owner/tenant, timestamps | Every business record belongs to one store; all reads/writes are scoped. |
| Product | ID, store ID, name, type, base unit, base-unit price, cost basis, stock, threshold, active state, timestamps | Non-negative price/cost/stock; valid product type; stock is base-unit quantity. |
| ProductUnit | ID, product ID, name, base quantity, selling price | Positive base quantity, non-negative price, unique unit name per product. |
| Customer | ID, store ID, name, phone/contact, email, active state, timestamps | Store-scoped; preserve historical references and debt records. |
| Sale | ID, store ID, customer, receipt/reference, idempotency key, status, total, paid, balance, payment method, actor, created time | Valid state transitions, no overpayment, no future timestamp, store-consistent customer. |
| SaleItem | ID, sale ID, product ID, selected unit ID/reference, unit/name snapshot, quantity, base quantity snapshot, base quantity deducted, sale-time price, line total, cost snapshot | Positive quantities; preserve historical values if catalog changes. |
| SalePayment | ID, sale ID, amount, method, type/status (payment or reversal), actor, timestamp, optional reference | Positive valid entries; append-only ledger; over-balance checks under transaction. |
| InventoryMovement | ID, store, product, movement type, quantity change, previous/new quantity, actor, reason, linked sale/purchase, timestamp | Movement and stock update commit together. |
| Alert | ID, store, type, product, message, resolution state, created/resolved timestamps | No duplicate open low-stock alert for the same product/type. |
| Purchase / PurchaseItem (P1) | Supplier/reference, status, creator/approver, items, quantity, unit cost, total, timestamps | Pending creation does not change stock; approval mutates stock once. |
| Notification (P1) | Recipient/store, title/message, event reference, read timestamp, created timestamp | Must be generated by defined events and scoped to valid recipients. |
| AuditLog | Actor, action, entity, entity ID, details, timestamp | Append-only to regular users; never stores secrets. |

### Source-of-truth rules

- Flask models and migrations should be the canonical schema if the recommended target architecture is adopted.
- Do not use both a mutable customer aggregate and a separate sale/payment ledger as independent truths. The ledger/sale records are authoritative; any cached customer outstanding balance must be updated in the same transaction and covered by reconciliation tests.
- Store product cost or sale-line cost snapshots sufficient to make historical profit reports stable when product cost changes later.
- Product deactivation must preserve sale history and foreign-key integrity.

## 10. API contract and integration approach

### 10.1 Target integration

**Recommended:** Next.js remains responsible for UI and presentation. Flask owns business rules, database writes, store authorization, sessions, and calculation logic. Next.js `/api/*` routes may remain as a thin backend-for-frontend (BFF) for same-origin requests, but must proxy/translate to the Flask contract rather than querying PostgreSQL or reimplementing sales/inventory logic.

This keeps the browser on one stable interface while preventing Next.js and Flask from making independent stock/payment decisions.

### 10.2 Canonical current server route families

The Flask server currently exposes these route families (exact prefixes depend on blueprint registration):

| Area | Current server capability observed |
|---|---|
| `/auth` | Register, login, current user, logout |
| `/product` | Create, get, list, update, delete, adjust stock, inventory movements |
| `/sales` | Create/list/get sale, defaults, quick-add last recorded sale item, receipt |
| `/customers` | List/create/update/delete, customers with balances, customer history |
| `/alerts` | List, generate low-stock, resolve, delete |
| `/dashboard` | Dashboard, daily summary, daily business brief |
| `/health` | Health response |

### 10.3 Capabilities exposed by the client but missing as equivalent Flask modules

At review time, the Next.js app also contains routes for purchases, debts/repayments, reports, notifications, store profile, and user administration. These must be moved into or explicitly supported by the canonical server before the UI is considered integrated. A direct database call from a Next.js route is not an acceptable substitute for an API capability on the selected backend.

### 10.4 Contract requirements

- Publish one machine-readable API contract (OpenAPI or equivalent) and generate or validate TypeScript request/response types against it.
- Use one product type vocabulary, one unit schema, one payment-method vocabulary, and one ID contract across client and server.
- Preserve the API's `store_id`/store context rules consistently.
- Keep error responses stable; clients must not have to parse server exception strings to identify a validation code.
- Bound list endpoints through pagination, explicit date filtering, or documented limits.
- Do not let the client authoritatively send calculated stock, total, balance, profit, or role claims.
- Any BFF must forward session/auth state safely and must not silently bypass Flask authorization.

### 10.5 Contract examples (illustrative target, not current wire format)

Create sale request:

```json
{
  "store_id": 1,
  "client_transaction_id": "client-generated-unique-id",
  "customer_id": 24,
  "payment_method": "Cash",
  "amount_paid": "2500.00",
  "items": [
    {"product_id": 17, "unit_id": 3, "quantity": 2},
    {"product_id": 31, "unit_id": 8, "quantity": 1, "unit_price": "500.00"}
  ]
}
```

The example does not require the client to send line totals or sale total. A service-line price override is permitted only for a service. The server resolves unit metadata and calculates the authoritative total.

Illustrative successful result:

```json
{
  "success": true,
  "data": {
    "sale": {
      "id": 1001,
      "receipt_no": "REC-...",
      "total_amount": "3000.00",
      "amount_paid": "2500.00",
      "balance": "500.00",
      "status": "COMPLETED"
    }
  }
}
```

Exact IDs, casing, wrapper shape, and field names must be finalized in the API contract; this example is a target illustration, not a claim about the current response body.

## 11. Current repository findings and gaps

The observations below are code-derived. “Target” items in this PRD are recommendations, not current behavior.

### 11.1 Client repository

- The app is a Next.js/React/TypeScript application using PostgreSQL through `pg`, with route handlers under `app/api/*`.
- It has pages for login, admin/manager/employee dashboards, sales/POS, inventory, transactions, customers, debts, purchases, reports, notifications, users, and settings.
- Its schema includes users, products, sales, sale items, purchases, purchase items, notifications, and audit logs. A migration adds customers, product type, units, customer-linked sales, amount paid, and sale payments. A separate migration adds store name.
- Login currently uses username/password and a seven-day signed JWT cookie. The first-admin setup endpoint is protected by `SETUP_KEY` and refuses setup when users already exist. The token reader verifies the signed claim but does not reload the current user record to re-check active status or updated role on each request.
- Dashboard page files pass a role constant into the shared dashboard component. The dashboard API itself requires a session but does not restrict the activity-feed payload by role. A user who opens a privileged dashboard URL directly may receive a privileged-looking UI, and the generic dashboard endpoint can expose recent audit activity to roles that should not see it. Enforce role checks at the page/API/data level.
- Product PATCH runs the same full-form validator as product creation, so a genuinely partial PATCH is not reliably supported despite the HTTP method. The contract must not imply partial-update semantics unless they are implemented.
- Product and sale routes perform writes directly from the Next.js API layer. That duplicates business responsibility if the Flask service is intended to be the backend.
- `GET /api/products` currently selects a limited set of product fields and does not return `item_type`, `base_unit`, or `low_stock_threshold` even though sales/inventory UI types and behaviors expect product-type and base-unit details. The UI can therefore misidentify services, including treating zero-stock services like out-of-stock physical products.
- `/api/dashboard` currently uses a hard-coded low-stock value of `5` rather than each product's configured threshold. That can disagree with alert generation and inventory UI. It also sums stock quantities across products that may use different base units, then labels the result “Stock Units”; the aggregate is not meaningful when, for example, pieces, grams, and millilitres are mixed.
- The reporting route constructs a start bound for daily/monthly/yearly periods but applies only `created_at >= start`; monthly/yearly results can include records after the requested period unless an upper bound is added. The trend query is always a rolling seven-day window regardless of the selected period, and the UI's yearly placeholder text does not describe a single coherent year definition.
- Client product migrations are separate SQL scripts, while setup instructions also refer to `db/schema.sql`. Fresh database initialization and upgrade order need to be made deterministic so required commerce/store columns and tables cannot be missed.
- The notifications API can read and mark notifications, but the reviewed code does not establish a complete event-generation workflow. Define event producers before treating notifications as a working capability.
- Sale cancellation currently restores physical stock for an unpaid sale and rejects cancellation if `amount_paid > 0`. This is a useful safety constraint, but the product still needs a complete documented reversal/refund policy, reason capture, and tests for duplicate cancellation.
- Client database records use UUIDs and do not consistently carry a `store_id`, while the Flask schema uses integer IDs and store-scoped records. A direct database/backend switch is unsafe without a deliberate data plan.

### 11.2 Server repository

- The server is a Flask application organized into routes, services, SQLAlchemy models, utilities, Alembic/Flask-Migrate migrations, and pytest tests.
- It has store-scoped authorization, product/service support, base units, product units with independent prices, sale payment/balance fields, customer outstanding balances, inventory movements, low-stock alerts, Lagos/UTC date utilities, and sale idempotency support through a client transaction ID.
- The sale service records selected unit and base quantity information and rejects insufficient stock. It also records inventory movements for physical sales and allows service-specific price overrides.
- The README documents a broad pytest suite covering authentication/configuration, products, units, sale-line pricing, sales, idempotency, migrations, customers/debts, alerts, and dashboards. This is a strong foundation, but passing tests must be verified in CI before release.
- The server currently authenticates by email/password using Flask sessions, whereas the client authenticates by username/password using a JWT cookie.
- The server role model and user-administration endpoints do not currently match the client ADMIN/MANAGER/EMPLOYEE behavior.
- The server exposes sales defaults and a quick-add endpoint that returns the most recently recorded sale item, but the current client POS screen does not use these server endpoints; instead it builds its own product/cart list.
- The server has no equivalent API modules for the client's purchase approval, payment repayment, notifications, user administration, store settings, or advanced reports at the reviewed revision.
- The server's existing customer aggregate balance increases when credit sales are created, but the reviewed server route set does not include a repayment endpoint to decrease that balance. Debt collection must be completed in the canonical backend before credit sales are released.
- The server currently has sale timestamp UTC/Lagos-day utilities and tests, while corresponding client reporting must use consistent date boundaries too.

### 11.3 Cross-repository integration blockers

1. Two database owners and two independent sets of business rules.
2. UUID-vs-integer identifiers and different table shapes.
3. Username/JWT-vs-email/Flask-session authentication.
4. Global user role vs store ownership/authorization mismatch.
5. `PRODUCT`/`SERVICE` vs `Physical`/`Service` values.
6. Different unit quantity precision and field names.
7. Client-only purchases, repayments, notifications, reports, users, and store settings.
8. Divergent customer-balance calculation and repayment support.
9. Product API response fields missing from the client UI contract.
10. Low-stock threshold inconsistency and reporting period-boundary defects.
11. Migration/bootstrap behavior not yet demonstrated as reproducible on a clean database and an existing database.

## 12. Non-functional requirements

### 12.1 Security and privacy

- All production traffic shall use HTTPS.
- Passwords shall be hashed with a suitable password-hashing algorithm and never logged or returned.
- Session cookies shall be HttpOnly, Secure in production, and configured consistently with the chosen same-origin/BFF or cross-origin architecture.
- Secrets shall be supplied through deployment environment/secret management and excluded from source control.
- Protected endpoints shall enforce roles and store scope on the server.
- All SQL shall use parameterized queries or ORM-bound values.
- Inputs shall be validated server-side; IDs, role values, price values, quantities, date filters, and enum strings must not be trusted merely because the client validates them.
- Rate limiting/login abuse protections should be introduced before broad public exposure.
- Audit logs must exclude credentials, tokens, and secrets.

### 12.2 Data integrity and recovery

- Sale creation, repayment, adjustment, purchase approval, and reversal must be atomic transactions.
- Database constraints shall enforce critical invariants as well as application validations.
- No ordinary operation may hard-delete a record needed for historical financial or stock records.
- Production data shall be backed up on a defined schedule; restore steps must be tested, not merely documented.
- Migrations must be forward-tested on a copy of existing data and tested from an empty database.

### 12.3 Performance

Target initial performance on a normal broadband/mobile connection and modest production deployment:

- Common list/dashboard requests: p95 under 1.5 seconds at expected MVP load, excluding cold-start/wake delays.
- Checkout response: p95 under 2 seconds for a typical cart of up to 20 lines, excluding network interruptions.
- Search responses should remain responsive and avoid one database request per item where a joined/batched query can be used.
- Long histories and reports must be bounded/paginated and backed by indexes for commonly filtered fields.

These are proposed product targets and should be adjusted from early production measurements.

### 12.4 Usability and accessibility

- Use the current dark charcoal/black and orange visual direction consistently.
- Show the naira symbol and format NGN values consistently.
- Every data-fetching screen needs loading, empty, success, and error states.
- Destructive or irreversible operations require confirmation and, where relevant, a reason.
- Forms must label inputs, show field-level validation where possible, preserve useful input after recoverable errors, and disable duplicate submission while a request is processing.
- POS must clearly distinguish physical stock from services and show the base quantity consequence of a chosen selling unit.
- Critical controls should be usable by keyboard and screen readers; status must not rely on color alone.

### 12.5 Maintainability

- One backend owns business rules and persistent writes.
- API contracts and schema migrations are version-controlled.
- Business logic is placed in services/domain functions rather than duplicated in React components and route handlers.
- Changes to financial or inventory rules require regression tests.
- Every implementation phase has explicit test instructions and acceptance checks.

## 13. Acceptance criteria and test matrix

Release is blocked if any P0 acceptance item fails.

| ID | Scenario | Expected result |
|---|---|---|
| AT-01 | Unauthenticated user requests products, sales, customers, or dashboard | API denies access and returns a stable 401-style error; no protected data leaks. |
| AT-02 | Employee calls user-management or manager-only report endpoint directly, or opens `/admin-dashboard` manually | Server returns forbidden or renders the employee-authorized view; no privileged activity data or mutation is exposed. |
| AT-03 | User A requests a product/customer/sale belonging to Store B | Server rejects the request; Store B data is not disclosed or changed. |
| AT-04 | Create a physical product with 100 pieces and carton unit of 40 pieces | Product stores 100 base units; carton has base quantity 40 and its own price. |
| AT-05 | Sell 2 cartons from 100 pieces | Sale records quantity 2, selected unit/carton, base quantity 40, base quantity deducted 80; stock becomes 20. |
| AT-06 | Attempt to sell 1 carton when only 39 pieces remain | Sale is rejected; no sale header/line/payment is committed and stock remains 39. |
| AT-07 | Two simultaneous requests each try to sell 1 carton from 40 pieces | At most one sale succeeds; final stock cannot be negative and only successful sale effects persist. |
| AT-07a | Submit a product PATCH with only its name or price | Either the documented partial-update behavior changes only that field, or the endpoint clearly rejects partial requests; no omitted field is silently reset to zero/default. |
| AT-08 | Sell a service with zero stock | Sale succeeds when all service validations pass; no stock deduction/movement occurs. |
| AT-09 | Override a service price at checkout | The actual price appears on the sale line/receipt and the catalog price remains unchanged. |
| AT-10 | Attempt to override a physical product price | Request is rejected unless an approved discount mechanism explicitly allows it. |
| AT-11 | Create a ₦1,000 sale and record ₦600 paid | Sale balance is ₦400; a customer is mandatory. |
| AT-12 | Create a sale with paid amount higher than calculated total | Request is rejected; no partial changes persist. |
| AT-13 | Retry a successful checkout with the same idempotency key | Existing sale is returned; stock/payment/customer balance are not applied twice. |
| AT-14 | Record a repayment equal to the remaining balance | Ledger adds payment, sale balance becomes zero, customer outstanding decreases correctly, and history remains. |
| AT-15 | Record a repayment larger than the remaining balance | Request is rejected under transaction; balances and ledger remain unchanged. |
| AT-16 | Manually remove more stock than is available | Request is rejected; stock and movement history remain unchanged. |
| AT-17 | Stock falls to or below product threshold | One unresolved low-stock alert exists and appears in alert/dashboard outputs without a manual generation step. |
| AT-18 | Stock rises above threshold | Open low-stock state is resolved; repeated operations do not create duplicate open alerts. |
| AT-19 | Create sale at 00:10 Africa/Lagos | Sale is included in the current Lagos day even if UTC date is the previous day. |
| AT-20 | Submit a future sale timestamp | Server rejects it or corrects it under the declared policy; future-dated sale does not enter metrics. |
| AT-21 | Compare dashboard daily total with completed sale history for the same Lagos day | Totals agree under the documented completed-sale definition. |
| AT-22 | Request a monthly report | Results include only records within that month, with Lagos day boundaries and an exclusive next-period upper bound. |
| AT-23 | Approve a pending purchase twice | Stock is increased once only; state and movement records remain consistent. |
| AT-24 | Cancel an unpaid completed sale | Sale remains in history as cancelled, physical stock is restored once, balance/audit data are corrected. |
| AT-25 | Attempt to cancel a sale with money paid but no recorded refund | Server blocks the action with a clear message until refund/reversal workflow is used. |
| AT-26 | Run migrations on a clean database | All required tables/columns exist and the app starts without manual undocumented SQL steps. |
| AT-27 | Upgrade a copy of the current client database | Data and relationships are preserved or mapped by the documented migration plan; row counts and balances reconcile. |
| AT-28 | API/database is unavailable at app load | UI shows a real connection/error state rather than implying the store has zero sales or zero stock. |
| AT-28a | Products use different base units (for example, pieces, grams, and millilitres) | Dashboard does not present their raw quantity sum as a meaningful “stock units” total; it groups quantities by base unit or omits the aggregate. |
| AT-29 | Generate receipt after catalog price/unit changes | Receipt retains historical sale-time unit and price snapshots. |
| AT-30 | Deactivate a user and reuse an old session | User loses protected access within the specified revocation window. |

## 14. Definition of done

A feature is not complete merely because the screen renders or the endpoint returns HTTP 200.

A P0 feature is done when:

1. Its behavior is specified in this document or an approved linked decision.
2. The server validates all inputs and authorization.
3. Database writes and associated effects are atomic.
4. Historical records remain accurate after catalog changes.
5. Success, empty, loading, and failure UI states are implemented.
6. Automated tests cover normal operation, invalid input, authorization, rollback, and duplicate submission where relevant.
7. Its migration works on a fresh database and an existing database copy.
8. The client/server contract is documented and exercised by an integration test.
9. Manual test steps are documented for the phase.
10. CI passes and production configuration/rollback implications have been reviewed.

## 15. Delivery plan

This plan follows the project's existing staged roadmap and prioritizes the operational spine over advanced functionality.

| Phase | Main outcome | Deliverables / exit gate |
|---|---|---|
| **Phase 0: Product and architecture** | Resolve product/technical ambiguities | Approve this PRD; choose canonical backend/auth contract; document roles, IDs, unit precision, payment ledger, cancellation rules, and deployment topology. |
| **Phase 1: Project and database foundation** | Reproducible app and schema | Backend/client environments; one migration owner; empty-db bootstrap; legacy-data mapping plan; seed data separated from production. |
| **Phase 2: Auth and access control** | Users access only their store and permitted modules | Canonical login/logout/current-user contract; ADMIN/MANAGER/EMPLOYEE server authorization; store-scoped integration tests; secure session revocation. |
| **Phase 3: Product and inventory** | Trustworthy catalog and stock | Product/service type, base unit, units/prices, opening stock, adjustment endpoints, stock movements, thresholds, migration tests. |
| **Phase 4: POS and sales** | Correct end-to-end sale | Cart line/unit selection, server totals, service price override, stock locking, idempotency, payment/balance/customer validation, receipt/history. |
| **Phase 5: Customer debt ledger** | Safe partial payments and collection | Customer search/history, “Who Owes Me,” payment ledger and repayment endpoint, reconciliation test, audit events. |
| **Phase 6: Alerts and Daily Business Brief** | Explain what needs attention today | Automatic low-stock synchronization, dashboard metrics, daily brief, Lagos date boundaries, unavailable comparison state. |
| **Phase 7: Reliability and usability** | Remove release blockers | Full test/bug-fix pass; role-gated pages; mobile-width POS check; loading/error states; cancellation safety; security review. |
| **Phase 8: Deployment preparation** | Production-ready release candidate | Production variables, HTTPS/session settings, migrations, backup/restore test, health checks, monitoring, release/rollback checklist. |
| **Phase 9: Deployment and monitoring** | Real controlled usage | Staged production rollout, smoke tests with test store data, verify sale/stock/debt/dashboard reconciliation, monitor failures and user confusion. |
| **Phase 10: Follow-on modules** | Expand beyond first release | Purchases/approval, detailed reports, event-driven notifications, export, expanded settings, then advanced capabilities based on usage. |

### Phase-by-phase test instructions

- **Architecture/schema:** apply migrations to a new database and a copy of existing data; compare table/column state; reconcile counts, stock, sales totals, and customer balances.
- **Authentication:** test correct/wrong credentials, logout, deactivation, direct URL access, cross-store access, role changes, and last-admin protection.
- **Inventory:** create product/service; create multiple units; adjust stock up/down; test negative stock rejection; confirm movement record and low-stock alert.
- **Sales:** run the acceptance cases for carton/base quantity, out-of-stock rejection, zero-stock service, price override, partial payment, customer requirement, concurrency, idempotency, and receipt snapshot.
- **Debt ledger:** record partial and final repayments; test overpayment, duplicate requests, cancellation/refund interaction, and customer total reconciliation.
- **Dashboard/brief:** compare API results manually against sale history for today/yesterday in `Africa/Lagos`, including a UTC/Lagos date-boundary sale and empty baseline.
- **Release candidate:** run all backend tests, client lint/build, integration tests against the canonical server, migration checks, health checks, backup/restore drill, and a production smoke test.

## 16. Deployment and data migration requirements

The cutover is a significant engineering task because both repositories currently write their own PostgreSQL-shaped schema.

### Required cutover sequence

1. Identify whether the deployed client database contains real shop data and list the exact source/target database environments. Do not assume the production database is empty.
2. Take a verified backup and record current row counts for users, products, sales, sale items, customers, sale payments, purchase records, notifications, and audit logs.
3. Approve the canonical schema and ID strategy. The current client uses UUID primary keys in its own schema; Flask models currently use integer primary keys. A mapping strategy is required if historical data must be preserved.
4. Write and test a one-time migration/import script for stores, users, products, units, customers, sales, sale lines, payment entries, purchases if in-scope, and audits where appropriate.
5. Preserve historical unit/price information and avoid recalculating prior transactions using current catalog prices.
6. Reconcile migrated stock to the source snapshot and validate every sale's `total = sum(lines)`, `balance = total − payments`, and customer outstanding total.
7. Apply Flask migrations in the deployed environment using a controlled release step.
8. Deploy the client BFF/API adapter against the canonical server. Stop writes to the legacy client-owned data routes so two databases cannot diverge.
9. Run role, store isolation, sale, stock, debt, receipt, alert, and daily summary smoke tests.
10. Retain a rollback path and preserve the source backup until the new system has passed the agreed observation period.

### Migration blockers that require an explicit decision

- Preserve UUIDs across the target model or map UUID-based client IDs to server integer IDs.
- Whether first-admin setup remains the provisioning path or account registration remains public.
- Whether usernames, emails, or both identify users.
- How to reconcile users whose role exists only in the client database with server stores/owners.
- How to import client sale payments and derive the server's customer outstanding balances without double-counting.
- How to map `PRODUCT`/`SERVICE` to `Physical`/`Service` and reconcile decimal-capable client unit conversions with integer server conversions.
- Whether purchases and notifications must migrate at first release or can remain in the legacy system until their P1 implementation.

No production cutover should proceed while these questions are being guessed inside code.

## 17. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Two backend implementations remain writable | Duplicate/conflicting stock, sales, and customer balances | Select one source of truth; move client routes to a BFF/proxy; prevent direct client DB writes after cutover. |
| ID and schema mismatch | Broken references or lost history | Build an explicit data mapping/import; test against a production backup; reconcile totals. |
| Role enforcement exists only in navigation | Employees can open privileged routes or read unauthorized aggregates | Add role checks in API and server page/layout guards; restrict dashboard activity data by role; test direct URLs and API calls. |
| Stale seven-day JWT role/active claim | Deactivated staff may keep access | Revalidate server-side sessions or use revocable sessions and revoke them on role/active changes. |
| Partial-update contract mismatch | Product edits fail or omitted values are reset unexpectedly | Define PATCH vs PUT semantics and test omission of each optional field. |
| Product API omits type/unit/threshold fields | Services appear unavailable; UI calculations become inconsistent | Make response schemas typed and contract-tested; include all required product fields. |
| Hard-coded low-stock value or mixed-unit stock sum | Dashboard disagrees with configured threshold/alerts or reports a meaningless aggregate | Calculate low stock from each product threshold; group quantities by base unit or omit the aggregate. |
| Reports lack upper date bound | Monthly/yearly totals are wrong | Use start-inclusive/end-exclusive periods in the store timezone. |
| Debt payments not supported by canonical server | Customer balances become stuck or drift | Implement one transactional repayment ledger before enabling credit sales in production. |
| No transaction idempotency in current client route | Double-tap/retry creates duplicate sale | Generate a stable idempotency key per checkout and enforce uniqueness server-side. |
| Sale cancellation without complete reversal logic | Stock, cash and customer debt mismatch | Reasoned audited cancellation; gate paid sales behind refund/reversal support. |
| Manual schema bootstrap differs from migrations | Fresh deployment fails or production schemas diverge | Adopt a single migration path and test fresh-install plus upgrade paths. |
| “Profit” interpreted as accounting profit without correct cost model | Misleading decisions | Label gross profit/estimates, persist sale-time cost, define service cost and inventory cost valuation. |
| Notifications table without event producers | Empty or misleading notification feature | Define events/recipients and implement transactional/outbox-style producers before shipping UI. |

## 18. Product decisions to record before engineering starts

These are not reasons to stop drafting the PRD. They are explicit decisions that need an owner and a written answer before the affected implementation/cutover proceeds.

| Decision | Recommended default | Why it matters |
|---|---|---|
| Authoritative backend | Flask server owns business logic and database; Next.js is UI/BFF | Avoids two independent write paths. |
| Production database | PostgreSQL | Both current implementations are built around PostgreSQL for production. |
| Authentication | One server-owned, revocable session contract; one canonical login identifier policy | Removes JWT-vs-Flask-session and username-vs-email conflicts. |
| V1 store access | One assigned store per user; store ID on every business record | Delivers tenant isolation without requiring a full multi-store switcher. |
| Product-type enum | One canonical wire enum, with a single mapping layer if labels differ | Prevents service/product misclassification. |
| Unit conversion precision | Whole-number base quantities in V1 | Matches current Flask unit model and stock quantity validation. Revisit if fractional kg/litre sales are required. |
| Balance source of truth | Payment ledger + sale totals; cached aggregates reconciled transactionally | Prevents inconsistent sale balance and customer debt totals. |
| Paid sale cancellation | Require explicit refund/reversal ledger before cancellation | Protects cash and debt reconciliation. |
| Quick Add semantics | Separate “repeat last recorded sale item” from “quick-add service shortcuts” and “frequent products” | The server already has distinct defaults and last-sale quick-add behavior; the UI should not conflate these actions. |
| First release boundary | Auth, inventory, sales, customer debt, low-stock alerts, dashboard/Daily Business Brief, receipts/history | Aligns with the staged roadmap and keeps the initial release focused on operational reliability. |
| Purchases/reports/notifications | P1 unless needed to support a validated launch customer | These are represented in the client but not yet matched by server APIs. |

## 19. Success metrics

The first release should be judged on operational correctness and time saved, not number of screens.

### Reliability metrics

- 100% of accepted physical sales have matching inventory deductions and movement records.
- 0 accepted sales that cause negative physical stock.
- 0 duplicate sales or stock deductions from replayed idempotency keys.
- 100% of completed sale balances reconcile with the payment ledger.
- 100% of dashboard/Daily Business Brief sales totals reconcile with completed sale records for the same time window.
- 100% of stock changes can be traced to a documented movement/source and actor, except explicitly system-generated opening/import events.
- 0 unauthorized cross-store reads/writes in automated authorization tests.

### Usability and adoption metrics

- A trained cashier can complete a typical sale without manually calculating line totals or stock conversions.
- A shop owner can identify today's sales and low-stock products from the dashboard without manually combining records.
- Measure median checkout completion time during pilot use and set a numeric target after a baseline is collected.
- Track failed checkout rate, repeat submissions, low-stock alert usefulness, and support issues during the first monitored release.

## 20. Source references and review notes

The findings in Section 11 are grounded in the repository contents reviewed on 12 October 2026.

### Client repository

- Repository: https://github.com/trexave-labs2/retailos-client
- README: https://github.com/trexave-labs2/retailos-client/blob/main/README.md
- PostgreSQL schema: https://github.com/trexave-labs2/retailos-client/blob/main/db/schema.sql
- Commerce migration: https://github.com/trexave-labs2/retailos-client/blob/main/db/migrations/20261008_commerce_features.sql
- Auth implementation: https://github.com/trexave-labs2/retailos-client/blob/main/lib/auth.ts
- Sale route: https://github.com/trexave-labs2/retailos-client/blob/main/app/api/sales/route.ts
- Product route: https://github.com/trexave-labs2/retailos-client/blob/main/app/api/products/route.ts
- Dashboard route: https://github.com/trexave-labs2/retailos-client/blob/main/app/api/dashboard/route.ts
- Report route: https://github.com/trexave-labs2/retailos-client/blob/main/app/api/reports/route.ts
- The client main branch included merged PR #4 (“WIP: customer management, debt ledger and commerce schema”) at the reviewed revision. Its PR description explicitly flagged checkout integration, service/unit wiring, migrations, and authorization/error handling as incomplete follow-up work.

### Server repository

- Repository: https://github.com/trexave-labs2/retailos-server
- README/API guide: https://github.com/trexave-labs2/retailos-server/blob/main/README.md
- Product service: https://github.com/trexave-labs2/retailos-server/blob/main/server/app/services/product_service.py
- Sales service: https://github.com/trexave-labs2/retailos-server/blob/main/server/app/services/sales_service.py
- Customer service: https://github.com/trexave-labs2/retailos-server/blob/main/server/app/services/customer_service.py
- Time utilities: https://github.com/trexave-labs2/retailos-server/blob/main/server/app/utils/time.py
- Unit migration: https://github.com/trexave-labs2/retailos-server/blob/main/migrations/versions/008_product_units.py
- Base-quantity migration: https://github.com/trexave-labs2/retailos-server/blob/main/migrations/versions/009_sale_base_quantity_deducted.py
- Security guidance: https://github.com/trexave-labs2/retailos-server/blob/main/docs/SECURITY.md

### Project roadmap reference

The project’s supplied roadmap puts architecture/data model before project setup, then database, authentication/access control, core entities, sales, alerts, Daily Business Brief, reporting, testing, UI polish, deployment preparation, deployment, and monitoring. It recommends getting the first release spine working through auth, inventory, sales, alerts, and the Daily Business Brief before taking on heavier intelligence features. The phased plan in Section 15 follows that sequencing.

---

## Final release rule

**Do not call RetailOS ready for real shop data until one authoritative backend owns every business write, migrations are repeatable, role/store authorization is tested, and sale → stock → payment/debt → receipt → dashboard records reconcile end to end.** A polished screen cannot compensate for a ledger that disagrees with itself.
