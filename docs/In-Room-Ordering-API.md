We’ll design a complete In-Room Ordering API that focuses on the real-world workflows of hotel room service, the friction points with traditional POS systems, and a clean separation between guest-facing APIs, kitchen operations, and hotel backend integration.

---

## 1. System Overview

A guest scans a QR code or opens the hotel app, authenticates using their **reservation ID + room number** (or last name). The API issues a **JWT tied to that reservation**. The guest can browse the menu, build a cart, and place an order.  

Once the order is placed:
- The API persists the order.
- Pushes a **webhook event** (e.g., `order.placed`) to registered kitchen endpoints (kitchen display system, printer relay, etc.).
- Optionally syncs with the **POS** or bypasses it for speed – here we highlight where POS integration breaks.

The kitchen can then send status updates (preparing, ready, delivered) via a staff-side API, which triggers further webhooks back to the guest app.

---

## 2. Tech Stack

- **Node.js + Express** – REST API
- **PostgreSQL** – relational store for reservations, orders, menu, webhook registrations
- **JWT** (jsonwebtoken) – stateless auth tied to `reservation_id`
- **Webhooks** – outbound HTTP POST to kitchen endpoints (with retry & signature)
- **pg** (node-postgres) – direct SQL for simplicity, or Knex for query building
- Optional: **Redis** for order event queue (reliable delivery)

---

## 3. Database Schema (PostgreSQL)

We design tables that mirror the hotel operation and avoid POS tight-coupling.

```sql
-- Guests & Reservations (managed externally by PMS, we sync minimal data)
CREATE TABLE reservations (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reservation_id VARCHAR(50) UNIQUE NOT NULL, -- e.g., "RES20235001"
    room_number   VARCHAR(10) NOT NULL,
    guest_name    VARCHAR(100),
    check_in      DATE NOT NULL,
    check_out     DATE NOT NULL,
    status        VARCHAR(20) DEFAULT 'checked_in',
    created_at    TIMESTAMPTZ DEFAULT now()
);

-- Menu categories & items
CREATE TABLE menu_categories (
    id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    sort_order INT DEFAULT 0
);

CREATE TABLE menu_items (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id   UUID REFERENCES menu_categories(id),
    name          VARCHAR(200) NOT NULL,
    description   TEXT,
    price         NUMERIC(10,2) NOT NULL,
    image_url     TEXT,
    available     BOOLEAN DEFAULT true,
    modifiers     JSONB,          -- e.g., [{"name":"Extra cheese","price":1.50}]
    dietary_tags  TEXT[],
    created_at    TIMESTAMPTZ DEFAULT now()
);

-- Orders
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reservation_id  UUID NOT NULL REFERENCES reservations(id),
    room_number     VARCHAR(10) NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending', -- pending, confirmed, preparing, ready, delivered, cancelled
    special_instructions TEXT,
    total_amount    NUMERIC(10,2),
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Order items
CREATE TABLE order_items (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id    UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    menu_item_id UUID REFERENCES menu_items(id),
    item_name   VARCHAR(200) NOT NULL,   -- snapshot
    quantity    INT DEFAULT 1,
    unit_price  NUMERIC(10,2) NOT NULL,
    modifiers   JSONB,                  -- selected modifier snapshots
    special_instructions TEXT
);

-- Webhook registrations (kitchen endpoints)
CREATE TABLE webhook_subscriptions (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event   VARCHAR(50) NOT NULL,        -- 'order.placed', 'order.status_changed', etc.
    url     TEXT NOT NULL,
    secret  VARCHAR(200),                -- for HMAC signature
    active  BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Order events for audit & webhook retry (if not using queue)
CREATE TABLE order_events (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id   UUID NOT NULL REFERENCES orders(id),
    event_type VARCHAR(50) NOT NULL,
    payload    JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

**Key design choices:**
- `order_items.item_name` and `unit_price` are snapshots to preserve pricing even if menu changes.
- `modifiers` inside `order_items` replicates selection at time of order.
- `order_events` table doubles as event store for reliable webhook delivery.

---

## 4. JWT Authentication Tied to Reservation

No guest usernames/passwords. The login endpoint accepts:

```json
{
  "reservation_id": "RES20235001",
  "room_number": "401",
  "last_name": "Smith"     // optional, to add security
}
```

If the posted data matches a `reservations` row with `status = 'checked_in'`, the API returns a JWT:

```javascript
const jwt = require('jsonwebtoken');

// Inside login route
const reservation = await db.query(
  `SELECT id, reservation_id, room_number FROM reservations 
   WHERE reservation_id = $1 AND room_number = $2 AND status = 'checked_in'`,
  [reservation_id, room_number]
);

if (!reservation.rows.length) return res.status(401).json({ error: 'Invalid credentials' });

const token = jwt.sign(
  {
    sub: reservation.rows[0].id,           // internal reservation UUID
    reservation_id: reservation.rows[0].reservation_id,
    room_number: reservation.rows[0].room_number,
    role: 'guest'
  },
  process.env.JWT_SECRET,
  { expiresIn: '12h' }                     // aligns with typical stay
);

res.json({ token });
```

**Middleware for protected routes:**

```javascript
function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  if (!token) return res.sendStatus(401);

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.sendStatus(403);
    req.user = user;  // contains reservation_id, room_number, role
    next();
  });
}
```

All guest-facing endpoints (`/menu`, `/cart`, `/orders`) require this middleware. The `reservation_id` embedded in the token ensures that a guest can only see their own orders.

---

## 5. API Endpoints

### 5.1 Guest-Facing (JWT protected)

| Method | Endpoint                  | Description |
|--------|---------------------------|-------------|
| POST   | `/auth/login`             | Login with reservation details → JWT |
| GET    | `/menu`                   | List categories with available items |
| GET    | `/menu/:itemId`           | Single item detail |
| POST   | `/cart`                   | Add item(s) to a pending order (cart) |
| GET    | `/cart`                   | Get current pending order for the guest |
| PUT    | `/cart/items/:itemId`     | Modify quantity/modifiers |
| DELETE | `/cart/items/:itemId`     | Remove item from cart |
| POST   | `/orders`                 | Place the order (converts cart → confirmed) |
| GET    | `/orders/:orderId`        | Order details & status |
| GET    | `/orders`                 | List all orders for this reservation (history) |
| POST   | `/orders/:orderId/cancel` | Cancel if still pending |

**Cart Mechanics:**  
A "cart" is simply an `order` with `status = 'pending'`. There can be at most one pending order per reservation. Adding items checks for existing pending order or creates one. This avoids a separate cart table while keeping data normalized.

### 5.2 Kitchen/Staff-Facing (authenticated via API key or separate JWT role)

| Method | Endpoint                    | Description |
|--------|-----------------------------|-------------|
| GET    | `/kitchen/orders`           | List all orders with status != delivered/cancelled (for kitchen display) |
| PUT    | `/kitchen/orders/:orderId/status` | Update order status (preparing, ready, delivered) |
| POST   | `/webhooks/subscribe`       | Register a new webhook endpoint (admin) |
| DELETE | `/webhooks/:id`             | Remove a subscription |

Status flow:  
`pending → confirmed → preparing → ready → delivered` (or `cancelled` at any point if allowed).

---

## 6. Kitchen Webhook Events

When an order passes through a significant state change, the API fires webhooks. Using the `webhook_subscriptions` table, we POST a signed payload to each subscribed URL.

**Core events:**

- `order.placed` – order created (status = confirmed)
- `order.status_changed` – any status update (payload includes old/new status)
- `item.ready` – optional per-item readiness
- `order.delivered` – final delivery

Payload example for `order.placed`:

```json
{
  "event": "order.placed",
  "timestamp": "2026-05-06T12:00:00Z",
  "data": {
    "order_id": "uuid",
    "room_number": "401",
    "items": [
      {
        "name": "Club Sandwich",
        "quantity": 2,
        "modifiers": [{"name": "No mayo"}],
        "special_instructions": "Cut in half"
      }
    ],
    "total_amount": 24.50,
    "special_instructions": "Knock loudly",
    "created_at": "..."
  }
}
```

**Delivery & Security:**  
- Add a `X-Webhook-Signature` header (HMAC-SHA256 of payload with subscription secret).
- Retry on failure (exponential backoff, up to 5 attempts) – using `order_events` as a log.
- The webhook receiver (kitchen display) acknowledges with `200 OK`. This decouples kitchen operations from the API’s main transaction.

Implementation snippet (simplified):

```javascript
async function fireWebhooks(eventType, payload) {
  const subs = await db.query(
    'SELECT * FROM webhook_subscriptions WHERE event = $1 AND active = true',
    [eventType]
  );
  subs.rows.forEach(sub => {
    const body = JSON.stringify({ event: eventType, timestamp: new Date(), data: payload });
    const signature = crypto.createHmac('sha256', sub.secret).update(body).digest('hex');
    // Post to sub.url with header x-webhook-signature: signature
    // Retry logic omitted for brevity
  });
}
```

---

## 7. Where POS Integration Breaks (and how we avoid it)

Traditional POS systems are often **pull-based**, synchronous, and tightly coupled to on-premise hardware. Room service ordering exposes these fractures:

| Breakpoint | Description | In Our Design |
|------------|-------------|---------------|
| **Menu Sync** | POS holds the menu, but guests need always-available digital menu. POS outages block menu display. | We maintain an independent `menu_items` table, served directly by the API. A background sync job from PMS/POS can update it, but the API never depends on POS for menu reads. |
| **Order Acceptance** | POS expects a terminal (waiter) to punch in orders. An API order might get lost if the POS is down or if the kitchen printer fails. | Orders are persisted first in our DB, then pushed asynchronously to POS (if integration exists). The webhook to kitchen ensures the order reaches display even if POS is down. |
| **Real-time Status** | POS often lacks granular status (e.g., "preparing", "ready") for room service, or requires manual update. | Our API provides explicit status transitions, exposed to kitchen staff app, and triggers real-time webhooks. The POS is notified later, not the source of truth. |
| **Payment Capture** | Room charge is typically posted to the guest folio in PMS. POS tries to handle this, but often fails if PMS link is broken. | We record the order with `total_amount`. Payment/posting to PMS is a separate asynchronous job (not part of this API scope), avoiding transactional coupling. |
| **Multi-device Kitchen** | POS bind to one kitchen printer/display. Webhooks allow many subscribers (main kitchen, bar, pantry) registered dynamically. | The webhook subscription table allows any number of endpoints per event type. |
| **Inventory** | POS may reject an order if inventory is zero, causing poor guest experience. | We allow ordering even if inventory is low (flag item as `available = false` only when manual decision made). Optional inventory system can be built independently. |

**Result:** The API is the **source of truth** for room service orders. POS and PMS become downstream systems, optional and asynchronous.

---

## 8. Typical F&B Order Workflow (Sequence)

1. Guest logs in → gets JWT.  
2. Guest browses `/menu` (fetched from `menu_items`, not POS).  
3. Guest builds cart via `/cart` endpoints (pending order in DB).  
4. Guest calls `POST /orders` → order status moves to `confirmed`, fire `order.placed` webhook.  
5. Kitchen display receives webhook; shows new order. No POS involved yet.  
6. Kitchen staff updates status (preparing, ready) using staff API; each change fires `order.status_changed` webhook (guest app can poll or receive push).  
7. On delivery, status set to `delivered`; final webhook.  
8. A separate service (not detailed here) can post the final order to the PMS for room charge, or to POS for reporting.

---

## 9. Quick Node.js Project Structure

```
in-room-ordering/
├── src/
│   ├── config/         (DB, env)
│   ├── models/         (query functions)
│   ├── middleware/      (auth, error handling)
│   ├── routes/
│   │   ├── auth.js
│   │   ├── menu.js
│   │   ├── orders.js
│   │   ├── kitchen.js
│   │   └── webhooks.js
│   ├── services/       (webhook dispatcher, etc.)
│   └── app.js
├── migrations/
└── package.json
```

Use `express.Router()` for clear separation. Example `orders.js` (guest placing order):

```javascript
router.post('/', authenticateToken, async (req, res) => {
  const client = await db.pool.connect();
  try {
    await client.query('BEGIN');
    // Find pending order for user
    const pending = await client.query(
      `SELECT id FROM orders WHERE reservation_id = $1 AND status = 'pending'`,
      [req.user.sub]
    );
    if (!pending.rows.length) return res.status(400).json({ error: 'No pending order (cart empty)' });

    // Update status to confirmed
    await client.query(
      `UPDATE orders SET status = 'confirmed', updated_at = now() WHERE id = $1`,
      [pending.rows[0].id]
    );

    // Fetch full order for webhook payload
    const order = await getOrderById(client, pending.rows[0].id); // assemble items
    await client.query('COMMIT');

    // Fire webhook asynchronously (don't block response)
    fireWebhooks('order.placed', order).catch(console.error);

    res.json({ message: 'Order placed', order_id: order.id });
  } catch (err) {
    await client.query('ROLLBACK');
    res.status(500).json({ error: 'Could not place order' });
  } finally {
    client.release();
  }
});
```

---

## 10. Summary

We built a **POS-independent room service API** that:
- Authenticates guests purely via their reservation (JWT).
- Holds its own menu and order life cycle.
- Uses webhooks to push events to kitchen displays/printers, decoupling from brittle POS hardware.
- Exposes clear breakpoints where legacy POS integration often fails, and shows how to avoid them.

This API can be extended with real-time guest notifications (WebSockets), delivery time estimates, upselling logic, and a PMS posting connector – all without altering the core ordering flow.
