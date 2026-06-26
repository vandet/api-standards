# Example — Orders

Business flow with state transitions, filtering, and bulk operations.

---

## List Orders

**Request**

```http
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
            "id": "ord_001",
            "user_id": 1,
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

```http
GET /api/v1/orders/ord_001
Authorization: Bearer {token}
Accept: application/json
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Order retrieved successfully.",
    "data": {
        "id": "ord_001",
        "user_id": 1,
        "status": "pending",
        "items": [
            { "id": 1, "product_id": 10, "name": "Laptop",  "price": 200.00, "quantity": 1 },
            { "id": 2, "product_id": 20, "name": "Mouse",   "price": 25.00,  "quantity": 2 }
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

---

## Create an Order

**Request**

```http
POST /api/v1/orders
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

```json
{
    "items": [
        { "product_id": 10, "quantity": 1 },
        { "product_id": 20, "quantity": 2 }
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
        "id": "ord_001",
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
        "items.1.product_id": ["Product #20 is out of stock."]
    }
}
```

---

## Cancel an Order

POST on a resource action — not a CRUD method.

**Request**

```http
POST /api/v1/orders/ord_001/cancel
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
        "id": "ord_001",
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

---

## Process Payment

**Request**

```http
POST /api/v1/orders/ord_001/pay
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
Idempotency-Key: 7f6b2a1c-3d4e-5f6a-7b8c-9d0e1f2a3b4c
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
        "id": "ord_001",
        "status": "completed",
        "payment_id": "pay_xyz789",
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

---

## Bulk Cancel Orders

**Request**

```http
POST /api/v1/orders/bulk-cancel
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "ids": ["ord_001", "ord_002", "ord_003"],
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
            { "id": "ord_001", "success": true },
            { "id": "ord_002", "success": true },
            { "id": "ord_003", "success": true }
        ]
    }
}
```

**Response — 207 Partial failure**

```json
{
    "success": false,
    "message": "Some orders could not be cancelled.",
    "code": "BULK_PARTIAL_FAILURE",
    "data": {
        "cancelled": 2,
        "failed": 1,
        "items": [
            { "id": "ord_001", "success": true },
            { "id": "ord_002", "success": false, "code": "ORDER_ALREADY_PAID", "message": "Order already paid." },
            { "id": "ord_003", "success": true }
        ]
    }
}
```
