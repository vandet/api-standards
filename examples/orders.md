# Example — Orders

Business flow with state transitions, filtering, and bulk operations.

---

## List Orders

**Request**

```
GET /api/v1/orders?status=pending&sort=-created_at&page=1&per_page=20&include=statuses,payment_methods
Authorization: Bearer {token}
Accept: application/json
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Orders retrieved successfully.",
    "data": [
        {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "user_id": "661f9511-f3ac-52e5-b827-557766551111",
            "status": "pending",
            "total_amount": 250.00,
            "currency": "USD",
            "payment_method": "card",
            "created_at": "2026-06-26T10:00:00Z"
        }
    ],
    "included": {
        "statuses": [
            { "value": "pending",    "label": "Pending" },
            { "value": "processing", "label": "Processing" },
            { "value": "completed",  "label": "Completed" },
            { "value": "cancelled",  "label": "Cancelled" }
        ],
        "payment_methods": [
            { "value": "card",   "label": "Credit / Debit Card" },
            { "value": "bank",   "label": "Bank Transfer" },
            { "value": "wallet", "label": "E-Wallet" }
        ]
    },
    "pagination": {
        "current_page": 1,
        "last_page": 3,
        "per_page": 20,
        "total": 47,
        "from": 1,
        "to": 20
    },
    "links": {
        "first": "/api/v1/orders?page=1",
        "last":  "/api/v1/orders?page=3",
        "next":  "/api/v1/orders?page=2",
        "prev":  null
    }
}
```

---

## Get an Order

**Request**

```
GET /api/v1/orders/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer {token}
Accept: application/json
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Order retrieved successfully.",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "user_id": "661f9511-f3ac-52e5-b827-557766551111",
        "status": "pending",
        "items": [
            { "id": "aaa00001-0000-0000-0000-000000000001", "product_id": "bbb00001-0000-0000-0000-000000000001", "name": "Laptop", "price": 200.00, "quantity": 1 },
            { "id": "aaa00002-0000-0000-0000-000000000002", "product_id": "bbb00002-0000-0000-0000-000000000002", "name": "Mouse",  "price": 25.00,  "quantity": 2 }
        ],
        "subtotal":       250.00,
        "discount":       0.00,
        "tax":            25.00,
        "total_amount":   275.00,
        "currency":       "USD",
        "payment_method": "card",
        "notes":          null,
        "created_at":     "2026-06-26T10:00:00Z",
        "updated_at":     "2026-06-26T10:00:00Z"
    }
}
```

**Response — 404 Not found**

```json
{
    "success": false,
    "message": "Order not found.",
    "code": "ORDER_NOT_FOUND",
    "errors": {}
}
```

---

## Create an Order

**Request**

```
POST /api/v1/orders
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
Idempotency-Key: 7f6b2a1c-3d4e-5f6a-7b8c-9d0e1f2a3b4c
```

```json
{
    "items": [
        { "product_id": "bbb00001-0000-0000-0000-000000000001", "quantity": 1 },
        { "product_id": "bbb00002-0000-0000-0000-000000000002", "quantity": 2 }
    ],
    "payment_method": "card",
    "notes": "Please deliver before 5pm."
}
```

**Response — 201 Created**

```json
{
    "success": true,
    "message": "Order created successfully.",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "status": "pending",
        "total_amount": 275.00,
        "currency": "USD",
        "created_at": "2026-06-26T10:00:00Z"
    }
}
```

**Response — 409 Item out of stock**

```json
{
    "success": false,
    "message": "One or more items are out of stock.",
    "code": "ORDER_ITEM_OUT_OF_STOCK",
    "errors": {
        "items.1.product_id": ["Product is out of stock."]
    }
}
```

**Response — 422 Validation failed**

```json
{
    "success": false,
    "message": "Validation failed.",
    "code": "VALIDATION_FAILED",
    "errors": {
        "items":          ["At least one item is required."],
        "payment_method": ["Payment method is required."]
    }
}
```

---

## Cancel an Order

POST on a resource action — not a CRUD method.
See [01-api-design.md — POST for Actions](../01-api-design.md#post-for-actions).

**Request**

```
POST /api/v1/orders/550e8400-e29b-41d4-a716-446655440000/cancel
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "reason": "Customer requested cancellation."
}
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Order cancelled successfully.",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "status": "cancelled",
        "cancelled_at": "2026-06-26T11:00:00Z"
    }
}
```

**Response — 409 Already paid**

```json
{
    "success": false,
    "message": "This order has already been paid and cannot be cancelled.",
    "code": "ORDER_ALREADY_PAID",
    "errors": {}
}
```

**Response — 404 Not found**

```json
{
    "success": false,
    "message": "Order not found.",
    "code": "ORDER_NOT_FOUND",
    "errors": {}
}
```

---

## Process Payment

**Request**

```
POST /api/v1/orders/550e8400-e29b-41d4-a716-446655440000/pay
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
Idempotency-Key: 9a8b7c6d-5e4f-3a2b-1c0d-9e8f7a6b5c4d
```

```json
{
    "payment_method": "card",
    "card_token": "tok_abc123"
}
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Payment processed successfully.",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "status": "completed",
        "payment_id": "ccc00001-0000-0000-0000-000000000001",
        "paid_at": "2026-06-26T10:05:00Z"
    }
}
```

**Response — 422 Payment failed**

```json
{
    "success": false,
    "message": "Payment was declined by the gateway.",
    "code": "PAYMENT_FAILED",
    "errors": {}
}
```

**Response — 409 Already paid**

```json
{
    "success": false,
    "message": "This order has already been paid.",
    "code": "ORDER_ALREADY_PAID",
    "errors": {}
}
```

---

## Bulk Cancel Orders

Bulk action on a collection. See [01-api-design.md — Bulk Actions](../01-api-design.md#bulk-actions).

**Request**

```
POST /api/v1/orders/bulk-cancel
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "ids": [
        "550e8400-e29b-41d4-a716-446655440000",
        "661f9511-f3ac-52e5-b827-557766551111",
        "772a0622-04bd-63f6-c938-668877662222"
    ],
    "reason": "System maintenance."
}
```

**Response — 200 All cancelled**

```json
{
    "success": true,
    "message": "3 orders cancelled successfully.",
    "data": {
        "cancelled": 3,
        "failed": 0,
        "items": [
            { "id": "550e8400-e29b-41d4-a716-446655440000", "success": true },
            { "id": "661f9511-f3ac-52e5-b827-557766551111", "success": true },
            { "id": "772a0622-04bd-63f6-c938-668877662222", "success": true }
        ]
    }
}
```

**Response — 207 Partial failure**

> `207` is the only case where `data` appears alongside `success: false`.
> The data describes which items succeeded and which failed.
> See [03-response-standard.md](../03-response-standard.md#207-exception).

```json
{
    "success": false,
    "message": "Some orders could not be cancelled.",
    "code": "BULK_PARTIAL_FAILURE",
    "data": {
        "cancelled": 2,
        "failed": 1,
        "items": [
            { "id": "550e8400-e29b-41d4-a716-446655440000", "success": true },
            { "id": "661f9511-f3ac-52e5-b827-557766551111", "success": false, "code": "ORDER_ALREADY_PAID", "message": "Order already paid." },
            { "id": "772a0622-04bd-63f6-c938-668877662222", "success": true }
        ]
    },
    "errors": {}
}
```
