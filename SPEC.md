# Conveyor — Phase 1 Specification

Northline Trade Supplies Ltd
Version 1.0 — 17 October 2026

## 1. Background

Northline is a B2B wholesaler of electrical and plumbing supplies, selling to tradespeople via 40 trade counters, a mobile app and a phone desk. Volume is approximately 12,000 orders per day, concentrated between 06:30 and 08:00.

The existing order system (OMS) processes each order as a single synchronous transaction: stock check, payment, order record, warehouse notification. This causes:

- Outages when the payment provider is slow, blocking all order intake.
- Stock left reserved when payment fails, with no release.
- No real-time revenue reporting; figures are produced by nightly batch.

OMS is to be replaced by an event-driven system on Kafka, named Conveyor.

## 2. Phase 1 objective

Deliver the order flow as three independent services communicating over HTTP. Kafka is not introduced in this phase. The objective is to fix the service boundaries, data ownership and business rules so that phase 2 replaces the transport without changing the logic.

Requirements:

- Runs locally with no cloud dependencies.
- Each service is independently buildable, runnable and deployable.
- Each service owns its data exclusively. No shared database.
- The order service is the only service exposed to clients (app, tills, phone desk).

## 3. Business rules

1. Stock reservation is all-or-nothing per order.
2. The order total is calculated by the order service from catalogue prices. Prices submitted by clients are ignored.
3. A failed payment rejects the order. There is no pending-payment state.
4. An order is charged at most once.
5. Every rejected order carries a reason.

## 4. Order flow

1. Order service receives the order and records it as `PLACED`.
2. Order service requests a reservation from the inventory service.
   - Rejected → order status `REJECTED` with reason. Stop.
   - Reserved → order status `STOCK_RESERVED`.
3. Order service requests payment from the payment service.
   - Failed → order status `REJECTED` with reason. Stop.
   - Paid → order status `CONFIRMED`.
4. Order service returns the final order to the client.

Known limitation, accepted for phase 1: a payment failure at step 3 leaves stock reserved. This reproduces the OMS defect and is resolved in phase 2 by the event-driven design. It must not be fixed in phase 1.

## 5. Services

### 5.1 Order service

Responsibility: accept orders from clients, coordinate the flow in §4, hold the order record.

Owns: orders and order lines.

Enforces: rules 2 and 5.

API (client-facing, agreed with the app and till teams; fixed):

`POST /orders`

Request:

```json
{
   "customerId": "cust-42",
   "items": [
      { "productId": "SKU-001", "quantity": 2 },
      { "productId": "SKU-002", "quantity": 1 }
   ]
}
```

Response `201`:

```json
{
   "orderId": "<uuid>",
   "customerId": "cust-42",
   "items": [
      { "productId": "SKU-001", "quantity": 2, "unitPrice": 49.99 },
      { "productId": "SKU-002", "quantity": 1, "unitPrice": 19.99 }
   ],
   "totalAmount": 119.97,
   "status": "CONFIRMED",
   "rejectionReason": null,
   "createdAt": "2026-10-17T07:12:04Z"
}
```

`status` is one of `PLACED`, `STOCK_RESERVED`, `CONFIRMED`, `REJECTED`. `rejectionReason` is null unless status is `REJECTED`.

`GET /orders/{orderId}` — returns the same representation. `404` if not found.

Validation: `customerId` required; `items` non-empty; `quantity` ≥ 1; unknown `productId` → `400`.

### 5.2 Inventory service

Responsibility: hold stock levels and product prices; decide whether an order can be reserved.

Owns: product catalogue (code, description, unit price) and stock on hand per product.

Enforces: rule 1.

API (internal):

`POST /reservations`

Request:

```json
{
   "orderId": "<uuid>",
   "items": [
      { "productId": "SKU-001", "quantity": 2 }
   ]
}
```

Response `200`, one of:

```json
{ "orderId": "<uuid>", "status": "RESERVED", "items": [ { "productId": "SKU-001", "quantity": 2, "unitPrice": 49.99 } ] }
```

```json
{ "orderId": "<uuid>", "status": "REJECTED", "reason": "Insufficient stock: SKU-004" }
```

On `REJECTED`, no stock is decremented for any line. On `RESERVED`, stock is decremented for every line and current unit prices are returned so the order service can compute the total.

`GET /stock/{productId}` — returns `{ "productId", "available" }`. Diagnostic.

Startup: load the catalogue in §6 if not already present.

### 5.3 Payment service

Responsibility: take payment for an order; record the outcome.

Owns: payments.

Enforces: rules 3 and 4.

API (internal):

`POST /payments`

Request:

```json
{ "orderId": "<uuid>", "amount": 119.97 }
```

Response `200`, one of:

```json
{ "paymentId": "<uuid>", "orderId": "<uuid>", "status": "PAID" }
```

```json
{ "paymentId": "<uuid>", "orderId": "<uuid>", "status": "FAILED", "reason": "Card declined" }
```

If a payment already exists for `orderId`, return it without charging again.

`GET /payments/{orderId}` — returns the payment for the order. `404` if none. Diagnostic.

Payment provider: simulated. Fails a configurable proportion of requests; default 0.2. Must be settable to 0 and 1 for testing.

### 5.4 Conventions

- Identifiers: UUID strings, generated by the owning service.
- Money: decimal, GBP, two decimal places, serialised as a JSON number.
- Timestamps: ISO-8601, UTC.
- Business outcomes (rejected, failed) are `200` responses with a status field. HTTP error codes are for invalid requests and system errors only.

## 6. Test catalogue

| productId | description | unitPrice | stock |
|---|---|---|---|
| SKU-001 | Keyboard | 49.99 | 10 |
| SKU-002 | Mouse | 19.99 | 25 |
| SKU-003 | Monitor | 199.00 | 3 |
| SKU-004 | Headset | 79.50 | 0 |
| SKU-005 | Webcam | 59.00 | 5 |

## 7. Acceptance criteria

All three services running.

| # | Scenario | Input | Expected |
|---|---|---|---|
| 1 | Normal order | 2 × SKU-001, 1 × SKU-002; failure rate 0 | `201`, `CONFIRMED`, total 119.97; SKU-001 stock 8 |
| 2 | Out of stock | 1 × SKU-004 | `201`, `REJECTED`, reason names SKU-004; no stock changed |
| 3 | All-or-nothing | 1 × SKU-001, 1 × SKU-004 | `201`, `REJECTED`; SKU-001 stock unchanged |
| 4 | Payment declined | 1 × SKU-002; failure rate 1 | `201`, `REJECTED`, reason "Card declined"; SKU-002 stock decremented (known limitation, §4) |
| 5 | Lookup | `GET /orders/{id}` for each above | Matching representation |
| 6 | Invalid request | empty `items` | `400` |

## 8. Out of scope

Kafka, warehouse notification, revenue reporting, authentication, real payment provider, containerisation, automated integration tests.

## 9. Roadmap

| Phase | Deliverable |
|---|---|
| 1 | Order flow over HTTP across three services |
| 2 | Kafka transport between services; stuck-reservation defect resolved |
| 3 | Payment retry and dead-letter handling; schema management |
| 4 | Revenue-per-minute stream for finance |
| 5 | Single-command local stack; integration test suite |