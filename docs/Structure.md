Here’s a detailed project structure for the In‑Room Ordering API, building on the earlier outline. It separates concerns cleanly and aligns with the described architecture (JWT auth, cart/order flow, webhooks, and PostgreSQL).

```
in-room-ordering/
├── package.json
├── .env                        # environment variables (DB_URL, JWT_SECRET, etc.)
├── .gitignore
├── README.md
│
├── src/
│   ├── app.js                  # Express app setup, middleware, route mounting
│   ├── server.js               # entry point: starts server after DB connection
│   │
│   ├── config/
│   │   └── index.js            # loads .env, exports configuration (DB pool, secrets)
│   │
│   ├── db/
│   │   ├── pool.js             # creates and exports pg Pool using config
│   │   └── migrations/         # raw SQL migration files (or use Knex)
│   │       ├── 001_create_reservations.sql
│   │       ├── 002_create_menu.sql
│   │       ├── 003_create_orders.sql
│   │       ├── 004_create_order_items.sql
│   │       ├── 005_create_webhook_subscriptions.sql
│   │       └── 006_create_order_events.sql
│   │
│   ├── middleware/
│   │   ├── authenticate.js     # JWT verification middleware
│   │   ├── authorizeRole.js    # checks JWT role (guest, kitchen, admin)
│   │   ├── errorHandler.js     # global error handling
│   │   └── validate.js         # request validation (using Joi or similar)
│   │
│   ├── models/                 # database query functions, no business logic
│   │   ├── reservation.js      # findReservation, checkIn, etc.
│   │   ├── menu.js             # getCategories, getItems, getItemById
│   │   ├── order.js            # createOrder, getCart, placeOrder, updateStatus, getById
│   │   ├── orderItem.js        # addItem, removeItem, updateItem (cart operations)
│   │   └── webhook.js          # getSubscriptions, createSubscription, deleteSubscription
│   │
│   ├── routes/
│   │   ├── auth.routes.js      # POST /auth/login
│   │   ├── menu.routes.js      # GET /menu, GET /menu/:itemId (JWT required)
│   │   ├── orders.routes.js    # guest cart & orders: /cart, /orders (JWT)
│   │   ├── kitchen.routes.js   # staff: GET /kitchen/orders, PUT /kitchen/orders/:id/status (role: kitchen)
│   │   └── webhook.routes.js   # admin: POST /webhooks/subscribe, DELETE /webhooks/:id
│   │
│   ├── services/               # business logic, outbound calls
│   │   ├── auth.service.js     # validates reservation, generates JWT
│   │   ├── order.service.js    # transactional logic for cart ↔ order, places order, updates status
│   │   ├── webhook.service.js  # fires webhooks: reads subscriptions, posts signed payload, retry logic
│   │   └── pos.service.js      # (optional) stub for future POS/PMS integration
│   │
│   └── utils/
│       ├── webhookSigner.js    # HMAC signature creation
│       ├── logger.js           # simple logger (winston or pino)
│       └── asyncHandler.js     # wraps async route handlers to catch errors
│
├── tests/                      # unit/integration tests
│   ├── auth.test.js
│   ├── menu.test.js
│   ├── orders.test.js
│   └── webhooks.test.js
│
└── docs/                       # API documentation (Swagger/OpenAPI spec)
    └── openapi.yaml
```

### Explanation of key parts

- **`config/`** – Centralised configuration; loads environment variables, creates database pool settings, JWT secret, etc.
- **`db/pool.js`** – A single PostgreSQL connection pool used by models.
- **`migrations/`** – Raw SQL files to version the database schema (can be run manually or with a tool like `node-pg-migrate`).
- **`middleware/`** – `authenticate.js` verifies the JWT and attaches `req.user`; `authorizeRole.js` restricts routes to specific roles (`guest` vs `kitchen` vs `admin`).
- **`models/`** – Pure data-access functions. They receive parameters and return query results, no HTTP concern. Example: `order.getCart(reservationId)` returns the pending order with items.
- **`services/`** – Contains the core business logic. For instance, `order.service.placeOrder()` handles the transaction: lock the pending order, change status to `confirmed`, commit, then call `webhook.service.fire('order.placed', order)`. This keeps routes thin.
- **`routes/`** – Define endpoints, apply middleware (auth, role, validation), call services, and send responses.
- **`utils/`** – Webhook signing, logging, and an `asyncHandler` to avoid repetitive `try/catch` in routes.
- **`app.js`** – Creates the Express app, attaches global middleware (`express.json()`, logger, etc.), mounts route modules (`app.use('/auth', authRoutes)`, etc.), and error handler.
- **`server.js`** – Imports `app`, connects to the database pool, and starts listening. Good separation for testing (you can import `app` without starting the server).

### How the pieces fit together (quick walkthrough)

1. A guest logs in via `auth.routes.js` → `auth.service.js` validates reservation with `reservation.js` model, returns a JWT.
2. The guest calls `GET /menu` → `menu.routes.js` (authenticated) → `menu.js` model queries DB → returns categories/items.
3. Adding to cart: `POST /cart` in `orders.routes.js` → `order.service.js` checks for existing pending order (model) or creates one, adds item via `orderItem.js` model.
4. Placing order: `POST /orders` → `order.service.placeOrder()` runs a transaction to confirm order, then fires `order.placed` webhook via `webhook.service.js`.
5. Webhook service loads all active subscriptions for that event from DB, signs the payload, sends HTTP request (with retries using `order_events` table as a log).
6. Kitchen staff updates status: `PUT /kitchen/orders/:id/status` (auth with `kitchen` role) → `order.service.updateStatus()` updates DB, fires `order.status_changed` webhook.

This structure is modular, testable, and closely follows the separation of concerns needed for a real-world hotel F&B system.