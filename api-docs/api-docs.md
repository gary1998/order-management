# 📦 Order Management Service API Documentation

## Overview

The **Order Management Service** handles order processing, fulfillment, tracking, and returns for the e-commerce platform.

* **Base URL:** `https://api.ecommerce.example.com/orders/v1`
* **Protocol:** HTTPS
* **Data Format:** JSON
* **Authentication:** JWT Bearer Token
* **Rate Limit:** 500 requests/minute per API key

---

## Authentication

All endpoints require JWT authentication.

### Header Format

| Header Name     | Value Example          |
| --------------- | ---------------------- |
| `Authorization` | `Bearer <JWT_TOKEN>`   |
| `X-API-Key`     | `<API_KEY>`            |

---

## Common HTTP Status Codes

| Status Code | Meaning                      |
| ----------- | ---------------------------- |
| `200`       | Success                      |
| `201`       | Order created                |
| `400`       | Bad request                  |
| `401`       | Unauthorized                 |
| `404`       | Order not found              |
| `409`       | Order state conflict         |
| `422`       | Validation error             |
| `500`       | Internal server error        |

---

# 🛒 Order APIs

## 1. Create Order

### Endpoint

```http
POST /orders
```

### Description

Creates a new order from cart items. Validates inventory and calculates totals.

### Request Body

| Field            | Type    | Required | Description                    |
| ---------------- | ------- | -------- | ------------------------------ |
| `userId`         | string  | Yes      | Customer user ID               |
| `items`          | array   | Yes      | Array of order items           |
| `shippingAddress`| object  | Yes      | Shipping address details       |
| `billingAddress` | object  | No       | Billing address (defaults to shipping) |
| `paymentMethod`  | string  | Yes      | Payment method ID              |
| `couponCode`     | string  | No       | Discount coupon code           |

### Example Request (cURL)

```bash
curl -X POST https://api.ecommerce.example.com/orders/v1/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "userId": "user_abc123",
    "items": [
      {
        "productId": "prod_7x8y9z",
        "quantity": 1,
        "price": 1799.99
      }
    ],
    "shippingAddress": {
      "name": "John Doe",
      "street": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "zipCode": "94105",
      "country": "US"
    },
    "paymentMethod": "pm_card_visa_4242",
    "couponCode": "SAVE10"
  }'
```

### Response Body

```json
{
  "orderId": "ord_xyz789abc",
  "userId": "user_abc123",
  "status": "pending_payment",
  "items": [
    {
      "productId": "prod_7x8y9z",
      "productName": "Dell XPS 15 Laptop",
      "quantity": 1,
      "unitPrice": 1799.99,
      "totalPrice": 1799.99
    }
  ],
  "subtotal": 1799.99,
  "discount": 180.00,
  "tax": 145.79,
  "shipping": 0.00,
  "total": 1765.78,
  "shippingAddress": {
    "name": "John Doe",
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zipCode": "94105",
    "country": "US"
  },
  "createdAt": "2024-01-16T10:00:00Z",
  "estimatedDelivery": "2024-01-20T23:59:59Z"
}
```

---

## 2. Get Order by ID

### Endpoint

```http
GET /orders/{orderId}
```

### Description

Retrieves detailed information about a specific order.

### Path Parameters

| Parameter | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| `orderId` | string | Yes      | Order ID    |

### Example Request (cURL)

```bash
curl -X GET https://api.ecommerce.example.com/orders/v1/orders/ord_xyz789abc \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Response Body

```json
{
  "orderId": "ord_xyz789abc",
  "userId": "user_abc123",
  "status": "shipped",
  "items": [
    {
      "productId": "prod_7x8y9z",
      "productName": "Dell XPS 15 Laptop",
      "quantity": 1,
      "unitPrice": 1799.99
    }
  ],
  "total": 1765.78,
  "trackingNumber": "1Z999AA10123456784",
  "carrier": "UPS",
  "shippedAt": "2024-01-17T14:30:00Z",
  "estimatedDelivery": "2024-01-20T23:59:59Z"
}
```

---

## 3. List User Orders

### Endpoint

```http
GET /users/{userId}/orders
```

### Description

Retrieves all orders for a specific user with pagination and filtering.

### Path Parameters

| Parameter | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| `userId`  | string | Yes      | User ID     |

### Query Parameters

| Parameter   | Type    | Required | Description                           |
| ----------- | ------- | -------- | ------------------------------------- |
| `status`    | string  | No       | Filter by order status                |
| `startDate` | string  | No       | Filter orders after date (ISO 8601)   |
| `endDate`   | string  | No       | Filter orders before date (ISO 8601)  |
| `page`      | integer | No       | Page number (default: 1)              |
| `limit`     | integer | No       | Results per page (default: 20)        |

### Example Request (cURL)

```bash
curl -X GET "https://api.ecommerce.example.com/orders/v1/users/user_abc123/orders?status=delivered&page=1&limit=10" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Response Body

```json
{
  "orders": [
    {
      "orderId": "ord_xyz789abc",
      "status": "delivered",
      "total": 1765.78,
      "createdAt": "2024-01-16T10:00:00Z",
      "deliveredAt": "2024-01-19T16:45:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "totalOrders": 45,
    "totalPages": 5
  }
}
```

---

## 4. Update Order Status

### Endpoint

```http
PATCH /orders/{orderId}/status
```

### Description

Updates the status of an order. Restricted to admin users and automated systems.

### Request Body

| Field    | Type   | Required | Description                                    |
| -------- | ------ | -------- | ---------------------------------------------- |
| `status` | string | Yes      | New status: processing, shipped, delivered, cancelled |
| `notes`  | string | No       | Status change notes                            |

### Example Request (cURL)

```bash
curl -X PATCH https://api.ecommerce.example.com/orders/v1/orders/ord_xyz789abc/status \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "status": "shipped",
    "notes": "Shipped via UPS Ground"
  }'
```

### Response Body

```json
{
  "orderId": "ord_xyz789abc",
  "previousStatus": "processing",
  "currentStatus": "shipped",
  "updatedAt": "2024-01-17T14:30:00Z"
}
```

---

## 5. Cancel Order

### Endpoint

```http
POST /orders/{orderId}/cancel
```

### Description

Cancels an order if it hasn't been shipped yet. Refunds payment and restores inventory.

### Request Body

| Field    | Type   | Required | Description           |
| -------- | ------ | -------- | --------------------- |
| `reason` | string | Yes      | Cancellation reason   |

### Example Request (cURL)

```bash
curl -X POST https://api.ecommerce.example.com/orders/v1/orders/ord_xyz789abc/cancel \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "reason": "Customer requested cancellation"
  }'
```

### Response Body

```json
{
  "orderId": "ord_xyz789abc",
  "status": "cancelled",
  "refundId": "ref_abc123xyz",
  "refundAmount": 1765.78,
  "cancelledAt": "2024-01-16T15:00:00Z"
}
```

---

## 6. Add Tracking Information

### Endpoint

```http
POST /orders/{orderId}/tracking
```

### Description

Adds shipping tracking information to an order.

### Request Body

| Field             | Type   | Required | Description                |
| ----------------- | ------ | -------- | -------------------------- |
| `trackingNumber`  | string | Yes      | Shipment tracking number   |
| `carrier`         | string | Yes      | Shipping carrier name      |
| `trackingUrl`     | string | No       | Tracking URL               |

### Example Request (cURL)

```bash
curl -X POST https://api.ecommerce.example.com/orders/v1/orders/ord_xyz789abc/tracking \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "trackingNumber": "1Z999AA10123456784",
    "carrier": "UPS",
    "trackingUrl": "https://www.ups.com/track?tracknum=1Z999AA10123456784"
  }'
```

### Response Body

```json
{
  "orderId": "ord_xyz789abc",
  "trackingNumber": "1Z999AA10123456784",
  "carrier": "UPS",
  "trackingUrl": "https://www.ups.com/track?tracknum=1Z999AA10123456784",
  "addedAt": "2024-01-17T14:30:00Z"
}
```

---

## 7. Get Order Tracking

### Endpoint

```http
GET /orders/{orderId}/tracking
```

### Description

Retrieves real-time tracking information for an order.

### Example Request (cURL)

```bash
curl -X GET https://api.ecommerce.example.com/orders/v1/orders/ord_xyz789abc/tracking \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Response Body

```json
{
  "orderId": "ord_xyz789abc",
  "trackingNumber": "1Z999AA10123456784",
  "carrier": "UPS",
  "status": "in_transit",
  "estimatedDelivery": "2024-01-20T23:59:59Z",
  "trackingEvents": [
    {
      "timestamp": "2024-01-17T14:30:00Z",
      "location": "San Francisco, CA",
      "status": "picked_up",
      "description": "Package picked up"
    },
    {
      "timestamp": "2024-01-18T08:15:00Z",
      "location": "Oakland, CA",
      "status": "in_transit",
      "description": "Package in transit"
    }
  ]
}
```

---

# 🔄 Return & Refund APIs

## 8. Create Return Request

### Endpoint

```http
POST /orders/{orderId}/returns
```

### Description

Creates a return request for an order or specific items.

### Request Body

| Field      | Type   | Required | Description                      |
| ---------- | ------ | -------- | -------------------------------- |
| `items`    | array  | Yes      | Items to return with quantities  |
| `reason`   | string | Yes      | Return reason                    |
| `comments` | string | No       | Additional comments              |

### Example Request (cURL)

```bash
curl -X POST https://api.ecommerce.example.com/orders/v1/orders/ord_xyz789abc/returns \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "items": [
      {
        "productId": "prod_7x8y9z",
        "quantity": 1
      }
    ],
    "reason": "defective_product",
    "comments": "Screen has dead pixels"
  }'
```

### Response Body

```json
{
  "returnId": "ret_abc123xyz",
  "orderId": "ord_xyz789abc",
  "status": "pending_approval",
  "items": [
    {
      "productId": "prod_7x8y9z",
      "quantity": 1,
      "refundAmount": 1799.99
    }
  ],
  "totalRefund": 1799.99,
  "returnLabel": "https://cdn.ecommerce.com/returns/ret_abc123xyz.pdf",
  "createdAt": "2024-01-21T10:00:00Z"
}
```

---

## 9. Get Return Status

### Endpoint

```http
GET /returns/{returnId}
```

### Description

Retrieves the status and details of a return request.

### Example Request (cURL)

```bash
curl -X GET https://api.ecommerce.example.com/orders/v1/returns/ret_abc123xyz \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Response Body

```json
{
  "returnId": "ret_abc123xyz",
  "orderId": "ord_xyz789abc",
  "status": "approved",
  "items": [
    {
      "productId": "prod_7x8y9z",
      "quantity": 1,
      "refundAmount": 1799.99
    }
  ],
  "totalRefund": 1799.99,
  "approvedAt": "2024-01-21T14:00:00Z",
  "returnDeadline": "2024-01-31T23:59:59Z"
}
```

---

## 10. Process Refund

### Endpoint

```http
POST /returns/{returnId}/refund
```

### Description

Processes a refund for an approved return. Admin only.

### Request Body

| Field          | Type    | Required | Description                    |
| -------------- | ------- | -------- | ------------------------------ |
| `refundAmount` | decimal | Yes      | Refund amount                  |
| `refundMethod` | string  | Yes      | original_payment, store_credit |

### Example Request (cURL)

```bash
curl -X POST https://api.ecommerce.example.com/orders/v1/returns/ret_abc123xyz/refund \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "refundAmount": 1799.99,
    "refundMethod": "original_payment"
  }'
```

### Response Body

```json
{
  "returnId": "ret_abc123xyz",
  "refundId": "ref_xyz789abc",
  "refundAmount": 1799.99,
  "refundMethod": "original_payment",
  "status": "refunded",
  "processedAt": "2024-01-25T10:30:00Z"
}
```

---

# 📊 Analytics & Reporting APIs

## 11. Get Order Statistics

### Endpoint

```http
GET /analytics/orders/stats
```

### Description

Retrieves order statistics for a date range. Admin only.

### Query Parameters

| Parameter   | Type   | Required | Description                  |
| ----------- | ------ | -------- | ---------------------------- |
| `startDate` | string | Yes      | Start date (ISO 8601)        |
| `endDate`   | string | Yes      | End date (ISO 8601)          |
| `groupBy`   | string | No       | Group by: day, week, month   |

### Example Request (cURL)

```bash
curl -X GET "https://api.ecommerce.example.com/orders/v1/analytics/orders/stats?startDate=2024-01-01&endDate=2024-01-31&groupBy=day" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Response Body

```json
{
  "period": {
    "startDate": "2024-01-01",
    "endDate": "2024-01-31"
  },
  "summary": {
    "totalOrders": 1247,
    "totalRevenue": 2456789.50,
    "averageOrderValue": 1970.23,
    "cancelledOrders": 43,
    "returnedOrders": 28
  },
  "dailyStats": [
    {
      "date": "2024-01-01",
      "orders": 45,
      "revenue": 88765.30
    }
  ]
}
```

---

## 12. Export Orders

### Endpoint

```http
POST /orders/export
```

### Description

Exports orders to CSV or JSON format. Admin only.

### Request Body

| Field       | Type   | Required | Description                    |
| ----------- | ------ | -------- | ------------------------------ |
| `format`    | string | Yes      | Export format: csv, json       |
| `startDate` | string | Yes      | Start date filter              |
| `endDate`   | string | Yes      | End date filter                |
| `status`    | string | No       | Filter by status               |

### Example Request (cURL)

```bash
curl -X POST https://api.ecommerce.example.com/orders/v1/orders/export \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "format": "csv",
    "startDate": "2024-01-01",
    "endDate": "2024-01-31",
    "status": "delivered"
  }'
```

### Response Body

```json
{
  "exportId": "exp_abc123",
  "status": "processing",
  "estimatedCompletion": "2024-01-16T11:00:00Z",
  "downloadUrl": null
}
```

---

## Error Response Format

All error responses follow this structure:

```json
{
  "errorCode": "ORDER_NOT_FOUND",
  "message": "Order with given ID does not exist",
  "timestamp": "2024-01-16T15:00:00Z",
  "requestId": "req_xyz123abc"
}
```

### Common Error Codes

| Error Code                | Description                          |
| ------------------------- | ------------------------------------ |
| `ORDER_NOT_FOUND`         | Order does not exist                 |
| `INVALID_ORDER_STATUS`    | Cannot perform action in current state |
| `INSUFFICIENT_INVENTORY`  | Product out of stock                 |
| `PAYMENT_FAILED`          | Payment processing failed            |
| `INVALID_ADDRESS`         | Shipping address validation failed   |
| `COUPON_INVALID`          | Coupon code is invalid or expired    |
| `CANNOT_CANCEL_ORDER`     | Order already shipped                |
| `RETURN_WINDOW_EXPIRED`   | Return period has expired            |

---
