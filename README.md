# Stock Level Management API - High-Level Specification

## Base URL
```
https://InterviewBrief.OurServer.com/api
```

## Overview
The Stock Level Management API allows selected customers to access real-time stock levels. Customers can either request stock information for a specific product using its SKU or retrieve a complete list of all available products and their stock levels.

## Authentication & Security
- The API uses **JWT (JSON Web Token)** for authentication and authorization.
- Only authorized customers can generate and use JWT tokens to access the API.
- Each request must include a valid JWT token in the `Authorization` header:
  ```http
  Authorization: Bearer <JWT_TOKEN>
  ```
- The JWT token contains the customer's identity and permissions, enabling role-based access control.
- Tokens have an expiration time to enhance security.
- Unauthorized requests receive a `401 Unauthorized` response.

## Backend Models
```csharp
public class Product
{
    public int ProductId { get; set; }
    public string SKU { get; set; }
    public int StockQuantity { get; set; }
}

public class Order
{
    public int OrderId { get; set; }
    public DateTime CreatedDate { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class OrderItem
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
}

public class PurchaseOrder
{
    public int PurchaseOrderId { get; set; }
    public int OrderId { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime? ExpectedDeliveryDate { get; set; }
    public DateTime? ActualDeliveryDate { get; set; }
    public OrderStatus Status { get; set; }
}

public class StockResponseDto
{
    public string SKU { get; set; }
    public int StockQuantity { get; set; }
    public DateTime? ExpectedDeliveryDate { get; set; }
}

public class ErrorResponse
{
    public string Error { get; set; }
}

public enum OrderStatus
{
    Created,
    Confirmed,
    PendingDelivery,
    Done,
    Cancelled
}

```

## API Endpoints

### 1. Authenticate & Obtain JWT Token
**Endpoint:**
```http
POST /api/auth/login
```
**Request Body:**
```json
{
  "username": "123@client.com",
  "password": "123ClientPassword"
}
```
**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR..."
}
```

### 2. Get Stock for a Specific Product (by SKU)
**Endpoint:**
```http
GET /api/stock/{sku}
```
**Headers:**
```http
Authorization: Bearer <JWT_TOKEN>
```
**Response (In Stock Example):**
```json
{
  "SKU": "098765qwerty",
  "StockQuantity": 15,
  "ExpectedDeliveryDate": null
}
```
**Response (Out of Stock Example):**
```json
{
  "SKU": "123456qwerty",
  "StockQuantity": 0,
  "ExpectedDeliveryDate": "2025-03-15T00:00:00Z"
}
```

### 3. Get Full Stock List
**Endpoint:**
```http
GET /api/stock
```
**Headers:**
```http
Authorization: Bearer <JWT_TOKEN>
```
**Response:**
```json
[
  { "SKU": "098765qwerty", "StockQuantity": 15, "ExpectedDeliveryDate": null },
  { "SKU": "123456qwerty", "StockQuantity": 0, "ExpectedDeliveryDate": "2025-03-15T00:00:00Z" }
]
```

### 4. Unauthorized Access Example
If a request is made with an invalid or missing JWT token, the API will return:
```http
HTTP/1.1 401 Unauthorized
```
```json
{
  "error": "Unauthorized. Invalid or expired token."
}
```

## Database Schema and Queries
### SQL Table Definitions with Indexing
```sql
CREATE TABLE Products (
    ProductId INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    StockQuantity INT NOT NULL
);

CREATE TABLE Orders (
    OrderId INT IDENTITY(1,1) PRIMARY KEY,
    CreatedDate DATETIME NOT NULL
);

CREATE TABLE OrderItems (
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    PRIMARY KEY (OrderId, ProductId),
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId),
    FOREIGN KEY (ProductId) REFERENCES Products(ProductId)
);

CREATE TABLE PurchaseOrders (
    PurchaseOrderId INT IDENTITY(1,1) PRIMARY KEY,
    OrderId INT NOT NULL,
    CreatedDate DATETIME NOT NULL,
    ExpectedDeliveryDate DATETIME NULL,
    ActualDeliveryDate DATETIME NULL,
    Status NVARCHAR(20) NOT NULL,
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId),
    INDEX idx_order_expected_delivery (OrderId, ExpectedDeliveryDate)
);
```

### Query to Fetch a Specific Product's Stock
```sql
SELECT
    p.SKU,
    p.StockQuantity,
    MIN(po.ExpectedDeliveryDate) AS ExpectedDeliveryDate
FROM Products p
LEFT JOIN OrderItems oi ON p.ProductId = oi.ProductId
LEFT JOIN Orders o ON oi.OrderId = o.OrderId
LEFT JOIN PurchaseOrders po ON o.OrderId = po.OrderId
WHERE p.SKU = '098765qwerty'
AND (po.ExpectedDeliveryDate IS NULL OR po.ExpectedDeliveryDate >= GETDATE())
AND po.Status IN ('Confirmed', 'PendingDelivery')
GROUP BY p.SKU, p.StockQuantity;
```

### Query to Fetch Full Stock List
```sql
SELECT 
    p.SKU, 
    p.StockQuantity, 
    MIN(po.ExpectedDeliveryDate) AS ExpectedDeliveryDate
FROM Products p
LEFT JOIN OrderItems oi ON p.ProductId = oi.ProductId
LEFT JOIN Orders o ON oi.OrderId = o.OrderId
LEFT JOIN PurchaseOrders po ON o.OrderId = po.OrderId
WHERE po.Status IN ('Confirmed', 'PendingDelivery')
AND (po.ExpectedDeliveryDate IS NULL OR po.ExpectedDeliveryDate >= GETDATE())
GROUP BY p.SKU, p.StockQuantity
ORDER BY p.StockQuantity DESC;

```

## Performance Considerations
- **Caching:**
  - Full stock list retrieval is cached (e.g., 5 minutes) using Redis or in-memory caching.
  - The API checks if the requested data is in the cache.
  - If found, cached data is returned instead of querying the database.
  - If not found, data is fetched from the database, stored in the cache, and returned.
  - Cached data expires after a set period or is removed when outdated.

- **Rate Limiting:**
  - The API limits the number of requests per user/IP (e.g., 100 requests per minute).
  - If exceeded, the API responds with:

```http
HTTP/1.1 429 Too Many Requests
```
```json
{
  "error": "Rate limit exceeded. Try again in 60 seconds."
}
```

