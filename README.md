# E-Commerce and Inventory Management System

**Backend Development Capstone, Option 4.** Designed, built and tested end to end by a single developer.

A production-style REST API with real-time updates. It covers authentication with JWT access and refresh tokens, role-based access control, product and inventory management, atomic stock updates at checkout, coupons, reviews, MongoDB aggregation analytics, a wishlist with recommendations, and an event-driven backend with Socket.IO.

---

## Features

- **Auth and RBAC:** register, login, `POST /api/auth/refresh`, `POST /api/auth/logout`. Roles are `customer`, `manager` and `admin`.
- **Products and inventory:** full CRUD, filtering by category, price range and keyword, sorting, pagination, `sku`, `stock`, `lowStockThreshold`, `averageRating` and `numReviews`.
- **Orders:** checkout validates stock, applies coupons, decrements stock atomically (no negative stock under concurrency) and restores stock on cancellation. Lifecycle: `pending -> confirmed -> shipped -> delivered`, or `cancelled`.
- **Coupons:** percentage and flat discounts, minimum order amount, maximum discount cap, expiry, global usage limit and single use per user.
- **Reviews:** one review per customer per product, verified-purchase flag, automatic rating recalculation.
- **Analytics (aggregation pipelines):** sales by date, sales and inventory by category, and an overview of KPIs.
- **Personalization:** wishlist and a recommendation engine that falls back to top-rated products for new users.
- **Events and real time:** native `EventEmitter` domain events decouple side effects, and Socket.IO pushes live updates to admins and customers.

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime / framework | Node.js (CommonJS), Express.js |
| Database | MongoDB with Mongoose ODM |
| Security | `jsonwebtoken`, `bcrypt`, `cookie-parser`, RBAC middleware |
| Events / real time | Native `events` (`EventEmitter`), `socket.io` |
| Utilities | `dotenv`, `cors`, `morgan` |

No other frameworks, unit-test libraries, containers or build tools are used. The API is tested with a Postman collection.

## Architecture

```
Client (browser demo / Postman)
   | HTTP                              | WebSocket
   v                                   v
Express: morgan > cors > json > cookieParser      Socket.IO (JWT handshake)
   |                                               rooms: admin-room, user:<id>
   v                                                   ^
Routes > protect (JWT) > authorize (RBAC)              |
   v                                                   |
Controllers --- eventBus.emit(...) ---> listeners.js --+
   v                                       |
Mongoose models <--- audit logs, rating recalculation
   v
MongoDB
```

Controllers only change data and emit a domain event. Everything else (audit log, notifications, low-stock checks, rating recalculation) runs in listeners, so an HTTP response never waits on side effects.

## Project Structure

```
ecommerce-inventory-system/
├── config/db.js
├── controllers/        auth, product, coupon, order, review, analytics, wishlist
├── events/             eventBus.js, listeners.js
├── middleware/         auth.js, role.js, errorHandler.js
├── models/             User, Product, Coupon, Order, Review, Wishlist, AuditLog
├── routes/             auth, product, coupon, order, review, analytics, wishlist, recommendation
├── seed/seedData.js
├── sockets/socketHandler.js
├── public/index.html   real-time demo console
├── postman_collection.json
├── server.js
├── package.json
├── .env.example
├── README.md
└── PROJECT_REPORT.md
```

## Getting Started

**Prerequisites:** Node.js 18.11 or newer, and MongoDB running locally on the default port.

```bash
# 1. Install dependencies
npm install

# 2. Create your environment file
cp .env.example .env        # on Windows: copy .env.example .env
# then set two long, different random strings for JWT_ACCESS_SECRET and JWT_REFRESH_SECRET

# 3. Load demo data (wipes the database first)
npm run seed

# 4. Start the server
npm run dev                 # or: npm start
```

Open **http://localhost:5000** for the demo console. The health check is at `http://localhost:5000/api/health`.

### Environment Variables

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `PORT` | no | `5000` | HTTP and WebSocket port |
| `NODE_ENV` | no | `development` | `production` hides stack traces, enables the secure cookie option and blocks seeding |
| `MONGO_URI` | yes | `mongodb://127.0.0.1:27017/ecommerce_inventory_db` | MongoDB connection string |
| `JWT_ACCESS_SECRET` | yes | none | Signs access tokens |
| `JWT_REFRESH_SECRET` | yes | none | Signs refresh tokens (must differ from the access secret) |
| `JWT_ACCESS_EXPIRES_IN` | no | `15m` | Access token lifetime |
| `JWT_REFRESH_EXPIRES_IN` | no | `7d` | Refresh token lifetime |
| `BCRYPT_SALT_ROUNDS` | no | `10` | bcrypt cost factor |
| `CLIENT_ORIGIN` | no | `http://localhost:5000` | Allowed CORS and Socket.IO origins (comma separated) |
| `COOKIE_SECURE` | no | `false` | Set `true` only when serving over HTTPS |

### Scripts

| Command | Action |
|---|---|
| `npm start` | Start the server |
| `npm run dev` | Start with automatic restart (`node --watch`) |
| `npm run seed` | Clear the database and load demo data |
| `npm run seed:destroy` | Clear the database and exit |

### Seeded Accounts

| Role | Email | Password |
|---|---|---|
| admin | `admin@shop.com` | `Admin@123` |
| manager | `manager@shop.com` | `Manager@123` |
| customer | `customer@shop.com` | `Customer@123` |

Four more customers (`priya@`, `rahul@`, `sneha@`, `arjun@example.com`) use the password `Customer@123`.

The seed creates 7 users, 26 products across 6 categories, 6 coupons, 50 orders spread over about 80 days, reviews generated from delivered orders and 5 wishlists. Five products start at or below their low-stock threshold.

**Seeded coupons:** `WELCOME10` (valid, min 500), `FLAT200` (min 1500), `BIGSALE25` (min 3000), `ONLYONE100` (one redemption in total), `EXPIRED20` (expired), `INACTIVE15` (inactive).

## Real-Time Demo Console

`public/index.html` is a single-file dark-themed console. It keeps a **Staff session** and a **Customer session** open at the same time, each with its own JWT and its own Socket.IO connection.

1. Click **Login as Admin** and **Login as Customer**. Both connection indicators turn green and show the rooms joined.
2. Click **Browse Products**, choose *Wireless Earbuds Pro* with quantity `3` and coupon `WELCOME10`, then click **Validate Coupon & Place Order**.
3. The admin log receives `order:new`, `stock:updated` and `alert:lowStock`. The customer log receives none of them.
4. Click **Update Order Status (Admin)**. The customer log receives `order:statusChanged`.
5. Click **Fetch Analytics & Recommendations** to see the dashboards and the customer's recommendations.

Click any log line to expand its JSON.

## API Reference

All endpoints are under `/api`. Protected endpoints need `Authorization: Bearer <accessToken>`.

### Auth

| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/auth/register` | Public | Create a customer account (role is always `customer`) |
| POST | `/auth/login` | Public | Returns `accessToken` and `refreshToken` (also sets an HTTP-only cookie) |
| POST | `/auth/refresh` | Public | Rotates the refresh token (cookie or `{ refreshToken }` body) |
| POST | `/auth/logout` | Public | Revokes the refresh token and clears the cookie |
| GET | `/auth/me` | Authenticated | Current user profile |

### Products

| Method | Route | Access | Description |
|---|---|---|---|
| GET | `/products` | Public | Query: `category`, `minPrice`, `maxPrice`, `search`, `inStock`, `minRating`, `sort` (`newest`, `price_asc`, `price_desc`, `rating`, `popularity`, `name_asc`, `name_desc`), `page`, `limit` |
| GET | `/products/low-stock` | Admin, Manager | Products at or below their threshold |
| GET | `/products/:id` | Public | Product details |
| POST | `/products` | Admin, Manager | Create a product |
| PUT | `/products/:id` | Admin, Manager | Update a product (stock is an absolute value) |
| DELETE | `/products/:id` | Admin, Manager | Soft delete (`isActive: false`) |

### Reviews

| Method | Route | Access | Description |
|---|---|---|---|
| GET | `/products/:productId/reviews` | Public | List reviews (`sort`: `newest`, `highest`, `lowest`; pagination) |
| POST | `/products/:productId/reviews` | Customer | Add a review `{ rating, title, comment? }` |
| PUT | `/reviews/:reviewId` | Review owner | Update a review |
| DELETE | `/reviews/:reviewId` | Owner, Admin | Delete a review |

### Coupons

| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/coupons/validate` | Authenticated | `{ code, items: [{ productId, quantity }] }` or `{ code, subtotal }` |
| GET | `/coupons` | Admin | List coupons (`isActive`, pagination) |
| POST | `/coupons` | Admin | Create a coupon |
| GET | `/coupons/:id` | Admin | Coupon details |
| PUT | `/coupons/:id` | Admin | Update a coupon (the code cannot change) |
| DELETE | `/coupons/:id` | Admin | Delete a coupon |

### Orders

| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/orders` | Authenticated | Checkout: `{ items, shippingAddress, paymentMethod?, couponCode? }` |
| GET | `/orders/my` | Authenticated | Own orders (`status`, pagination) |
| GET | `/orders` | Admin, Manager | All orders (`status`, `userId`, `startDate`, `endDate`, pagination) |
| GET | `/orders/:id` | Owner, Admin, Manager | Order details |
| PATCH | `/orders/:id/status` | Admin, Manager | `{ status, note? }` following the lifecycle |
| PATCH | `/orders/:id/cancel` | Owner, Admin, Manager | Cancel a pending or confirmed order; restores stock and releases the coupon |

### Analytics (Admin, Manager)

| Method | Route | Description |
|---|---|---|
| GET | `/analytics/sales-by-date` | `startDate`, `endDate`, `groupBy=day\|month`: revenue, orders, discounts, average order value |
| GET | `/analytics/sales-by-category` | Units, gross and net revenue, stock, inventory valuation, low-stock count |
| GET | `/analytics/overview` | Total revenue, orders, top 5 products, coupon stats, orders by status |

Analytics count every order that is not `cancelled`, and dates are grouped in UTC.

### Wishlist and Recommendations

| Method | Route | Access | Description |
|---|---|---|---|
| GET | `/wishlist` | Customer | Current wishlist |
| POST | `/wishlist/:productId` | Customer | Add (201 added, 200 already present) |
| DELETE | `/wishlist/:productId` | Customer | Remove |
| GET | `/recommendations` | Authenticated | Personalized or fallback recommendations (`limit`, max 20) |

## Response Format

Success: `{ "success": true, "message"?, "data": ... }`. Lists add `count` and `pagination: { page, limit, total, pages }`.

Error: `{ "success": false, "message", "code"?, "errors"? }`.

| Status | Codes |
|---|---|
| 400 | `VALIDATION_ERROR`, `INVALID_ID`, `INVALID_JSON`, `COUPON_INVALID`, `INVALID_STATUS_TRANSITION` |
| 401 | `NO_TOKEN`, `TOKEN_INVALID`, `TOKEN_EXPIRED`, `USER_NOT_FOUND`, `INVALID_CREDENTIALS`, `NO_REFRESH_TOKEN`, `REFRESH_TOKEN_INVALID`, `REFRESH_TOKEN_EXPIRED`, `REFRESH_TOKEN_REUSED` |
| 403 | `FORBIDDEN`, `ACCOUNT_DEACTIVATED` |
| 404 | `PRODUCT_NOT_FOUND`, `ORDER_NOT_FOUND`, `COUPON_NOT_FOUND`, `REVIEW_NOT_FOUND`, `NOT_IN_WISHLIST`, `ROUTE_NOT_FOUND` |
| 409 | `INSUFFICIENT_STOCK`, `DUPLICATE_KEY`, `REVIEW_EXISTS`, `COUPON_UNAVAILABLE`, `STATUS_CONFLICT` |
| 413 | `PAYLOAD_TOO_LARGE` |

## Socket.IO Reference

Connect with a valid access token: `io(url, { auth: { token } })` or `?token=...` in the query string. Admins and managers join `admin-room`. Every user joins `user:<userId>`.

| Event (server to client) | Room | Payload |
|---|---|---|
| `socket:ready` | the connecting socket | `{ userId, name, role, rooms, socketId, connectedAt }` |
| `order:new` | `admin-room` | `{ orderId, userId, customerName, totalAmount, discountAmount, couponCode, itemCount, status, createdAt }` |
| `stock:updated` | `admin-room` | `{ productId, name, sku, stock, lowStockThreshold, reason, orderId?, timestamp }` |
| `alert:lowStock` | `admin-room` | `{ productId, name, sku, stock, lowStockThreshold, message, timestamp }` |
| `order:statusChanged` | `user:<userId>` | `{ orderId, previousStatus, status, note, updatedAt }` |

### Domain Events (EventEmitter)

| Event | Emitted by | Listener actions |
|---|---|---|
| `order:placed` | Checkout | Audit log, `order:new`, `stock:updated`, low-stock threshold check |
| `order:statusUpdated` | Status change or cancel | Audit log, `order:statusChanged`, `stock:updated` after a cancel |
| `inventory:lowStock` | Order listener, product create or update | Audit log (warning), `alert:lowStock` |
| `review:updated` | Review create, update or delete | Recalculate `averageRating` and `numReviews`, audit log |
| `coupon:redeemed` | Checkout | Audit log |

## Testing with Postman

1. Import `postman_collection.json` into Postman (v2.1 format). It carries its own variables: `baseUrl`, `accessToken`, `refreshToken`, `productId`, `orderId` and several helpers.
2. Run `npm run seed`, then start the server.
3. Run the whole collection with the Collection Runner, in order. It has 75 requests, and every request carries `pm.test` assertions.

The suite covers auth and token rotation, RBAC failures (401 and 403), coupon validation, checkout with stock decrement and restore, reviews, analytics, wishlist and recommendations.

**Reset the data (`npm run seed`) before every full run.** The suite redeems the single-use `WELCOME10` coupon, places orders and creates reviews.

Command line (optional): `npx newman run postman_collection.json`.

## Key Design Decisions

- **Atomic stock:** `findOneAndUpdate({ stock: { $gte: qty } }, { $inc: { stock: -qty } })` is a single atomic operation, so overselling is impossible. Earlier lines are rolled back if a later line fails.
- **No multi-document transactions:** a standalone local MongoDB does not support them, so the code uses conditional single-document updates and compensating rollbacks.
- **Refresh tokens:** stored hashed (SHA-256), rotated on every use, revoked on logout, and reuse triggers revocation of all sessions.
- **Roles are read from the database** on every request, so a role change or deactivation takes effect immediately.
- **Order price snapshots** keep history and analytics correct after a product changes.
- **Soft delete for products** keeps orders, reviews and wishlists valid.

## Troubleshooting

| Problem | Fix |
|---|---|
| `Missing required environment variables` | Create `.env` from `.env.example` and set both JWT secrets |
| `MongoDB connection failed` | Start MongoDB locally and check `MONGO_URI` |
| `Port 5000 is already in use` | Change `PORT` in `.env` |
| Socket shows "Authentication error" | The access token expired. Use **Refresh token** in the demo console |
| Postman coupon tests fail on a second run | Run `npm run seed` again |

## License

ISC