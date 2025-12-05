
# E-Commerce System Design — Complete Structure, Low-Level Design, Schemas & Backend API List

**Generated:** 2025-12-04 (Asia/Kolkata timezone)

---

## Table of Contents
1. Overview
2. High-level architecture
3. Microservices and responsibilities
4. Database schemas (SQL CREATE TABLE statements)
5. Complete Backend API list (per service) — endpoints, DTOs, examples
6. DTO / Example JSON payloads
7. Low-Level Design: flows & sequence diagrams (text)
8. Caching, Search, Indexing & Performance tuning
9. Deployment & DevOps notes (Docker, K8s)
10. Security & Operational concerns
11. Appendix: naming conventions & helpful notes

---

## 1. Overview
This document contains a full system design for an e-commerce website intended to be production-capable and interview-ready. It includes microservice decomposition, complete database schemas (relational), API endpoints with request/response shapes, and low-level flows for critical features like product CRUD, cart, checkout, payment and inventory management.

---

## 2. High-level architecture
- Client: Web (React/Angular/Vue), Mobile (iOS/Android)
- API Gateway / Load Balancer
- Microservices:
  - User Service
  - Product Service
  - Cart Service
  - Order Service
  - Inventory Service
  - Payment Service
  - Notification Service
  - Search Service (Elasticsearch)
- Datastores: Relational DB (Postgres/MySQL) per service or logical DBs, Redis for cache & session, Elasticsearch for product search.
- Message Queue: Kafka/RabbitMQ for async events (order.created, payment.completed, inventory.updated)
- CDN for static assets & product images

---

## 3. Microservices & Responsibilities

### User Service
- Register/Login (JWT)
- Password reset
- Profile & address management
- Roles (USER, ADMIN)

### Product Service
- Product CRUD
- Category, brand management
- Product images (store URLs)
- Publish/unpublish products

### Cart Service
- Manage user carts (persisted in Redis or DB)
- Merge carts (guest → user on login)

### Inventory Service
- Track stock
- Reserve stock during checkout (soft reserve)
- Handle concurrent stock updates

### Order Service
- Create orders (SAGA pattern)
- Manage order lifecycle: CREATED → CONFIRMED → PACKED → SHIPPED → DELIVERED → CANCELLED → REFUNDED
- Interact with Payment Service and Inventory Service

### Payment Service
- Integrate with third-party payment gateways
- Handle webhooks
- Store payment transactions

### Notification Service
- Email and SMS notifications
- Templates for order confirmation, shipment, OTPs

### Search Service
- Index products into Elasticsearch
- Provide search + filter + facet APIs

---

## 4. Database Schemas (SQL)

### 4.1 User Service schema (Postgres / MySQL)
```sql
-- users table
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id VARCHAR(64) UNIQUE NOT NULL, -- e.g., "u_12345"
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20),
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(255),
  role VARCHAR(20) DEFAULT 'USER',
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE user_addresses (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  label VARCHAR(50), -- "home", "work"
  line1 VARCHAR(255),
  line2 VARCHAR(255),
  city VARCHAR(100),
  state VARCHAR(100),
  postal_code VARCHAR(20),
  country VARCHAR(100),
  phone VARCHAR(20),
  is_default BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### 4.2 Product Service schema
```sql
CREATE TABLE categories (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) UNIQUE,
  parent_id BIGINT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  product_id VARCHAR(64) UNIQUE NOT NULL, -- e.g., "p_12345"
  name VARCHAR(255) NOT NULL,
  sku VARCHAR(100) UNIQUE,
  description TEXT,
  price DECIMAL(12,2) NOT NULL,
  price_currency CHAR(3) DEFAULT 'INR',
  brand VARCHAR(255),
  category_id BIGINT,
  weight_kg DECIMAL(8,3),
  dimensions VARCHAR(255),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (category_id) REFERENCES categories(id)
);

CREATE TABLE product_images (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  product_id BIGINT NOT NULL,
  url VARCHAR(1024) NOT NULL,
  alt_text VARCHAR(255),
  is_primary BOOLEAN DEFAULT FALSE,
  sort_order INT DEFAULT 0,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

CREATE TABLE product_variants (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  product_id BIGINT NOT NULL,
  sku VARCHAR(100) UNIQUE,
  attributes JSON, -- e.g., {"color":"red","size":"M"}
  price DECIMAL(12,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);
```

### 4.3 Cart Service schema (Redis recommended, fallback DB model)
```sql
CREATE TABLE carts (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cart_items (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  cart_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  variant_id BIGINT,
  quantity INT NOT NULL DEFAULT 1,
  added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (cart_id) REFERENCES carts(id) ON DELETE CASCADE
);
```

**Redis structure (recommended):**
- Key: `cart:{user_id}` → value: JSON or a hash of `{variantId: quantity}`

### 4.4 Inventory Service schema
```sql
CREATE TABLE inventory (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  product_id BIGINT NOT NULL,
  variant_id BIGINT,
  stock_count INT NOT NULL DEFAULT 0,
  reserved_count INT NOT NULL DEFAULT 0,
  threshold INT DEFAULT 0, -- low stock threshold
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(id)
);
```

### 4.5 Order Service schema
```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_id VARCHAR(64) UNIQUE NOT NULL, -- e.g., "ord_20251204_0001"
  user_id BIGINT NOT NULL,
  total_amount DECIMAL(12,2) NOT NULL,
  currency CHAR(3) DEFAULT 'INR',
  status VARCHAR(50) DEFAULT 'CREATED',
  payment_status VARCHAR(50) DEFAULT 'PENDING',
  shipping_address_id BIGINT,
  billing_address_id BIGINT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  variant_id BIGINT,
  quantity INT NOT NULL,
  unit_price DECIMAL(12,2) NOT NULL,
  total_price DECIMAL(12,2) NOT NULL,
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

CREATE TABLE payments (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_id BIGINT NOT NULL,
  payment_provider VARCHAR(50),
  provider_transaction_id VARCHAR(255),
  amount DECIMAL(12,2),
  status VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

### 4.6 Notification / Audit tables (optional)
```sql
CREATE TABLE notifications (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  type VARCHAR(50),
  payload JSON,
  status VARCHAR(20) DEFAULT 'PENDING',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. Complete Backend API List

> Each endpoint includes: HTTP method, path, auth, request DTO, response DTO, example.

### 5.1 Authentication & User Service APIs
**Base:** `/api/users`

1. `POST /api/users/register`
- Auth: Public
- Request (UserRegisterDTO):
```json
{
  "userId": "dipankar",
  "email": "dipankar@example.com",
  "password": "P@ssw0rd",
  "fullName": "Dipankar Ghosh",
  "phone": "9876543210"
}
```
- Response: `201 Created` `{ "userId": "u_12345", "message": "User registered" }`

2. `POST /api/users/login`
- Auth: Public
- Request:
```json
{ "email": "dipankar@example.com", "password": "P@ssw0rd" }
```
- Response: `200 OK`
```json
{
  "accessToken": "jwt.token.here",
  "refreshToken": "refresh.token",
  "expiresIn": 3600
}
```

3. `POST /api/users/refresh-token`
- Auth: Refresh token required
- Request: `{ "refreshToken": "..." }`
- Response: new access token.

4. `POST /api/users/reset-password`
- Auth: Public (OTP/email flow) or authenticated
- Request (UserResetPasswordDto):
```json
{
  "userId": "u_12345",
  "oldPassword": "old",
  "newPassword": "new"
}
```
- Response: `200 OK` or `400` on mismatch.

5. `GET /api/users/{userId}`
- Auth: User or Admin
- Response: user profile

6. `POST /api/users/{userId}/addresses`
- Add address (see schema for fields)

7. `PUT /api/users/{userId}/addresses/{addressId}`

8. `DELETE /api/users/{userId}/addresses/{addressId}`

---

### 5.2 Product Service APIs
**Base:** `/api/products`

1. `POST /api/products` — Create product
- Auth: Admin
- Request (ProductDTO):
```json
{
  "productId": "p_123",
  "name": "T-Shirt",
  "sku": "TSHIRT-RED-M",
  "description": "Cotton T-shirt",
  "price": 499.00,
  "brand": "BrandX",
  "categoryId": 12,
  "variants": [
    {"sku":"TSHIRT-RED-M","attributes":{"color":"red","size":"M"},"price":499}
  ],
  "images": ["https://cdn.example.com/p1.jpg"]
}
```
- Response: `201 Created` with created productId.

2. `PUT /api/products/{productId}` — Update product
- Auth: Admin
- Request: Partial ProductDTO (only fields to update)
- Response: `200 OK`

3. `GET /api/products/{productId}` — Get product
- Auth: Public
- Response: ProductDTO with variant list & images.

4. `DELETE /api/products/{productId}` — Delete (soft or hard)
- Auth: Admin

5. `GET /api/products?category=...&brand=...&page=1&size=20&sort=price_asc`
- Auth: Public
- Response: Paginated list with facets.

6. `GET /api/products/search?q=shirt&filters=...`
- Auth: Public
- Uses Elasticsearch

7. `POST /api/products/{productId}/images` — Upload image URL
- Auth: Admin

8. `POST /api/products/{productId}/publish` — Mark product active
- Auth: Admin

---

### 5.3 Cart Service APIs
**Base:** `/api/carts`

1. `GET /api/carts` — Get current user's cart
- Auth: User
- Response:
```json
{
  "userId": "u_123",
  "items": [
    {"productId":"p_1","variantId": "v_1", "quantity": 2, "unitPrice": 499}
  ],
  "totalAmount": 998
}
```

2. `POST /api/carts/items` — Add or update item
- Auth: User
- Request:
```json
{ "productId": "p_1", "variantId":"v_1", "quantity": 2 }
```
- Response: `200 OK` updated cart

3. `DELETE /api/carts/items/{itemId}` — Remove item
- Auth: User

4. `POST /api/carts/merge` — Merge guest cart with user cart (on login)

---

### 5.4 Inventory Service APIs
**Base:** `/api/inventory`

1. `GET /api/inventory/{productId}` — Get stock info
- Auth: Public (or internal)
- Response:
```json
{ "productId": "p_1", "variantId":"v_1", "stockCount": 50, "reserved": 2 }
```

2. `POST /api/inventory/reserve` — Reserve stock (used during checkout)
- Auth: Internal (Order Service)
- Request:
```json
{ "orderId": "ord_1", "items":[{"productId":"p_1","variantId":"v_1","quantity":2}] }
```
- Response: success/failure (partial reserve allowed? define policy)

3. `POST /api/inventory/release` — Release reserved stock (on cancel/failure)

4. `POST /api/inventory/adjust` — Admin adjust (restock)

---

### 5.5 Order Service APIs
**Base:** `/api/orders`

1. `POST /api/orders` — Create order
- Auth: User
- Request:
```json
{
  "userId": "u_123",
  "cartId": "c_123",
  "shippingAddressId": 45,
  "billingAddressId": 45,
  "paymentMethod": "PAYMENT_GATEWAY",
  "couponCode": "NEWUSER10"
}
```
- Flow (high-level):
  - Validate cart & prices
  - Create transactionally `order` record with status `CREATED`
  - Call Inventory Service to reserve stock
  - Call Payment Service to create payment
  - On payment success, mark order `CONFIRMED` and deduct stock (or rely on inventory to deduct upon success)
  - Emit order.created event

2. `GET /api/orders/{orderId}` — Get order details

3. `GET /api/orders` — List user orders (paginated)

4. `POST /api/orders/{orderId}/cancel` — Cancel order (policy-based)

5. `POST /api/orders/{orderId}/refund` — Refund (admin or payment webhook)

---

### 5.6 Payment Service APIs
**Base:** `/api/payments`

1. `POST /api/payments/initiate` — Initiate payment (returns paymentUrl or client token)
- Auth: User
- Request:
```json
{ "orderId":"ord_1", "amount": 999.00, "currency":"INR", "paymentProvider":"RAZORPAY" }
```
- Response: `{ "paymentUrl":"https://...", "paymentId":"pay_123" }`

2. `POST /api/payments/webhook` — Webhook for gateway (public endpoint but validate signature)
- Auth: Gateway signature validation

3. `GET /api/payments/{paymentId}` — Payment status

---

### 5.7 Notification Service APIs
**Base:** `/api/notifications` (internal)

1. `POST /api/notifications/send` — Send email/SMS
- Auth: Internal
- Request:
```json
{ "userId": "u_123", "type":"ORDER_CONFIRMATION", "payload": {...} }
```

---

## 6. DTOs & Example JSON Payloads

### ProductDTO (example)
```json
{
  "productId": "p_123",
  "name": "Men's T-Shirt",
  "description": "100% Cotton",
  "sku": "TSHIRT-001",
  "price": 499.00,
  "brand": "BrandX",
  "categoryId": 12,
  "variants": [
    {
      "variantId":"v_1",
      "sku":"TSHIRT-RED-M",
      "attributes":{"color":"red","size":"M"},
      "price":499,
      "stock": 50
    }
  ],
  "images": ["https://cdn.example.com/p1.jpg"],
  "isActive": true
}
```

### OrderCreateDTO (example)
```json
{
  "userId": "u_123",
  "cartId": "c_123",
  "shippingAddressId": 45,
  "billingAddressId": 45,
  "paymentMethod": "RAZORPAY"
}
```

---

## 7. Low-Level Design & Sequence Flows

### 7.1 Add Product (Admin)
1. Client (Admin UI) → API Gateway → Product Service POST `/api/products`
2. Product Service validates DTO → writes to Product DB → index in Elasticsearch (async via event)
3. Respond `201 Created`

### 7.2 Browse & Search Products
1. Client → GET `/api/products` (list) => data served from Product Service.
2. For search (`/api/products/search?q=tee`), Product Service queries Elasticsearch and returns paginated results.
3. Cache popular queries in Redis (TTL 5-15 min).

### 7.3 Add to Cart
1. Client → Cart Service `POST /api/carts/items`
2. Cart Service stores in Redis (fast), optionally persist snapshot to DB.
3. Return updated cart.

### 7.4 Checkout / Place Order (SAGA pattern - simplified)
1. Client → Order Service `POST /api/orders`
2. Order Service:
   - Reads cart from Cart Service
   - Validates prices, taxes, coupons
   - Creates Order in DB with status `CREATED` (transactional in Order DB)
   - Calls Inventory Service `/reserve` (RPC)
   - If reserve succeeds:
       - Calls Payment Service `/payments/initiate` → return payment url/params
   - If payment completed (via webhook):
       - Payment Service notifies Order Service → Order Service marks `CONFIRMED`
       - Inventory Service deducts reserved stock (or mark reserved -> deducted)
       - Notification Service sends confirmation
   - If any step fails (reserve/payment), use compensating actions: release reservations and set order `FAILED` or `CANCELLED`.

### 7.5 Inventory concurrency
- Use optimistic locking (version column) or DB-level row locks.
- Prefer using Redis for high-concurrency flash sales (decrement atomic counters).
- For strong consistency, allocate stock via DB transaction + `SELECT ... FOR UPDATE`.

---

## 8. Caching, Search & Indexing

### Caching
- Redis for session tokens, carts, hot product details.
- Cache product detail pages and category lists. Use cache invalidation on update (publish/unpublish).
- Use CDN for product images & static resources.

### Search
- Elasticsearch index mapping for product fields:
  - name (text + keyword)
  - description (text)
  - price (double)
  - brand (keyword)
  - category (keyword)
  - attributes (nested)
- Use analyzers for language-specific tokenization.

### Indexing & DB performance
- Important indexes:
  - products(product_id), products(name), products(category_id), products(sku)
  - orders(order_id, user_id)
  - inventory(product_id, variant_id)
- Use composite indexes for common queries (user_id + created_at).

---

## 9. Deployment & DevOps Notes

### Containerization
- Package each service as Docker image.
- Use multi-stage builds, small base images (distroless/alpine where possible).

### Orchestration
- Kubernetes with one namespace per environment (dev/staging/prod)
- Use Horizontal Pod Autoscaler (HPA) based on CPU & custom metrics (queue length).

### Secrets & Config
- Use Vault or Kubernetes Secrets for DB passwords, JWT secrets, API keys.
- ConfigMaps for non-sensitive configs.

### Observability
- Centralized logging (ELK / EFK stack)
- Metrics (Prometheus + Grafana)
- Tracing (Jaeger or Zipkin)
- Alerting on error rates, latency, and inventory threshold.

---

## 10. Security & Operational Concerns

### Authentication & Authorization
- JWT with short expiry and refresh tokens.
- Use OAuth2 for external login (Google/Facebook).
- Role-based access control (Admin endpoints protected).

### Data Security
- Hash passwords with bcrypt/argon2.
- Use TLS for all network traffic.
- PCI-DSS compliance: do not persist raw card data; use payment gateway tokens.

### Rate limiting & Bot protection
- API Gateway for rate limiting
- WAF for filtering common attacks

### Backup & DR
- Regular DB backups, cross-region replication for production.
- Test restore procedures.

---

## 11. Appendix: Naming Conventions & Notes
- IDs: use prefixed IDs for readability (`u_`, `p_`, `ord_`) but keep DB numeric PKs for joins.
- Use API versioning: `/api/v1/...`
- DTO vs Entities: Keep DTOs lean; map to entities in service layer (ModelMapper / MapStruct).
- Error response convention:
```json
{ "timestamp":"2025-12-04T15:00:00Z", "status":400, "error":"Bad Request", "message":"Validation failed", "path":"/api/products" }
```

---

## End of Document

This file includes everything you asked for: structure, low-level design, complete schemas, and full backend API list.  
If you'd like, I can:
- Convert this markdown into a `.docx` or `.pdf` and include diagrams (UML PNGs).
- Generate Swagger (OpenAPI) YAML for these endpoints.
- Provide Dockerfiles, Kubernetes manifests, and CI/CD pipeline templates.

