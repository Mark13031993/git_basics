# Charkha Creations — Online Store (MERN)

## Context

Charkha Creations is a handmade ladies-suit business. The repo (`C:\GIT\Charkha`) currently contains only a README — this is a from-scratch build of the shop's first online store. Confirmed with the user: MERN stack, India-only (INR), Razorpay + Cash on Delivery at checkout, Cloudinary for product images, and a v1 feature set covering catalog, cart/checkout, customer accounts, and an admin panel to run day-to-day operations (products, stock, orders).

The goal of this plan is a working, verifiable v1: a customer can browse suits, add to cart, check out with either payment method, and see their order history; the shop owner can log in as admin to add/edit products with photos, track stock, and move orders through their fulfillment stages.

## Repository Structure

npm workspaces monorepo:
```
Charkha/
├── package.json          # workspaces: ["client","server"], root `npm run dev` via concurrently
├── .gitignore
├── client/                # Vite + React
│   └── src/{routes,components,features,context,lib,assets}
└── server/                # Express
    └── src/{config,models,controllers,routes,middleware,seed,utils}
```

## Package Choices

**Server**: express, mongoose, jsonwebtoken, bcryptjs, cookie-parser, cors, dotenv, cloudinary, razorpay, express-async-handler, express-validator, morgan (dev), nodemon (dev). No multer — images upload browser→Cloudinary directly (signed).

**Client**: vite + react + react-router-dom v6, @tanstack/react-query v5 (server state — products, orders, cart), React Context+useReducer for auth/cart (not Redux — too small to justify), axios (`withCredentials: true`), Tailwind CSS, react-hot-toast, react-hook-form.

**Auth transport**: httpOnly cookie (`sameSite: 'lax'`, `secure` in prod), not localStorage — avoids XSS token theft.

## MongoDB Schemas (`server/src/models/`)

- **User.js**: name, email (unique), password (bcrypt hash, `select:false`), role (`customer`|`admin`), phone, addresses[]. `pre('save')` hashes password; `matchPassword()` instance method.
- **Product.js**: name, slug (unique), description, category (enum of suit types), price, discountPrice?, images[{url, public_id}], sizes[{size, stock}], fabric, color, isFeatured, isActive. Virtual `inStock` from sizes.
- **Order.js**: user ref, orderItems[] (snapshot name/image/size/price/qty — never re-read live price), shippingAddress, paymentMethod (`COD`|`RAZORPAY`), paymentResult{razorpay ids/signature}, itemsPrice/shippingPrice/totalPrice, isPaid/paidAt, orderStatus (`pending→confirmed→shipped→delivered`, or `cancelled`), statusHistory[].
- **Cart.js**: separate collection, one doc per user (`user` unique), items[{product, name, image, size, price, qty}]. Kept separate from User so frequent cart writes don't rewrite the whole user doc; re-validated against live Product price/stock at checkout.

## API Routes (base `/api`, cookie auth via `protect`/`admin` middleware)

- `auth`: POST /register, /login, /logout · GET /me (private) · PUT /me (private)
- `products`: GET / , GET /:slug (public) · POST /, PUT /:id, DELETE /:id (admin)
- `upload`: GET /signature (admin) — Cloudinary signed-upload params
- `cart` (all private): GET /, POST /, PUT /:itemId, DELETE /:itemId, DELETE /
- `orders`: POST / (private, creates order — COD finalizes, RAZORPAY returns razorpay_order_id) · GET /mine (private) · GET /:id (owner/admin) · POST /:id/verify-payment (private) · GET / (admin, filterable) · PUT /:id/status (admin)

No public admin-signup route — first admin created via `server/src/seed/createAdmin.js` reading `ADMIN_EMAIL`/`ADMIN_PASSWORD` from env.

## Payment Flow — COD now, Razorpay deferred

**Scope change (confirmed with user):** v1 ships with Cash on Delivery only. Razorpay is deferred to a later follow-up once the user has a Razorpay account. The `Order` schema still carries `paymentMethod`/`paymentResult`/`isPaid` fields so adding Razorpay later is additive, not a rework — but the checkout UI only offers COD, and there is no Razorpay Checkout.js/verify-payment code yet (avoiding a half-built payment path).

1. Client POSTs `/api/orders` (paymentMethod: COD) → server re-validates cart against live product price/stock (never trusts client price), creates `Order` with `isPaid:false`, `orderStatus:'confirmed'` directly, decrements `Product` stock, clears the user's `Cart`.
2. When Razorpay is added later: `POST /api/orders` (paymentMethod: RAZORPAY) creates the order `isPaid:false` and calls `razorpayInstance.orders.create()`, client opens Checkout.js, `POST /api/orders/:id/verify-payment` recomputes the HMAC-SHA256 signature server-side before flipping `isPaid`/`orderStatus` — never trust client-reported success. Flagged here so the schema/route shape already anticipates it.

## Cloudinary Flow (signed direct upload)

Admin form requests `GET /api/upload/signature` (admin-protected) → server signs `{timestamp, folder}` with `CLOUDINARY_API_SECRET` via `cloudinary.utils.api_sign_request` → client POSTs the file + signature straight to Cloudinary's API (bypassing our server) → gets back `{secure_url, public_id}` → stored on the product on save. On product delete, server calls `cloudinary.uploader.destroy(public_id)` to avoid orphaned assets. Chosen over unsigned presets (avoids open upload endpoint) and over multer-proxied upload (no server bandwidth/disk cost).

## Frontend Routes

`/`, `/products` (?category=&search=), `/products/:slug`, `/cart`, `/checkout` (protected), `/login`, `/register`, `/account`, `/account/orders`, `/account/orders/:id`, `/admin`, `/admin/products`, `/admin/products/new`, `/admin/products/:id/edit`, `/admin/orders`, `/admin/orders/:id`. `ProtectedRoute`/`AdminRoute` wrappers for UX-level gating (real enforcement is server middleware).

Key components: Navbar (cart count, account menu), ProductCard/Grid, SizeSelector (reused in PDP + admin form), AddressForm (reused in checkout + account), OrderStatusBadge, AdminProductForm, AdminOrdersTable. Cart is a dedicated page, not a drawer, for v1 simplicity.

## Phased Build Order

1. **Scaffolding + auth**: workspaces, Vite client, Express server, Mongo connection, User model, register/login/logout/me, protect/admin middleware, cookie config, AuthContext, Login/Register pages, admin seed script.
2. **Product catalog**: Product model, admin CRUD routes, Cloudinary signature endpoint + upload flow, storefront listing/detail pages, admin product list/form.
3. **Cart + checkout (COD)**: Cart model/routes/context, Order model, COD order creation, checkout page, order confirmation, order history/detail pages. (Razorpay deferred — see Payment Flow section.)
4. **Admin order management**: admin order list (filterable) + status update route/UI, simple admin dashboard (counts: today's orders, low stock, revenue).
5. **Polish**: express-validator on mutating routes, centralized error middleware, frontend validation/toasts/loading/empty states, out-of-stock UI, mobile responsiveness pass, README setup docs.

Each phase should be independently verifiable before moving to the next — don't build checkout/payments before auth is confirmed working end-to-end.

## User-Provided Prerequisites

These need accounts/values from the user and can't be generated — flag and pause if a phase can't be fully tested without them yet (scaffolding can still proceed):

| Needed for | Variable(s) | Source |
|---|---|---|
| Phase 1 | `MONGODB_URI` | Local MongoDB, or free MongoDB Atlas cluster |
| Phase 1 | `JWT_SECRET` | Any long random string |
| Phase 1 | `ADMIN_EMAIL` / `ADMIN_PASSWORD` | User's choice for the seed script |
| Phase 2 | `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` | Free cloudinary.com account dashboard |
| Phase 3 | `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` (test mode) | Free razorpay.com account, Test Mode API Keys |
| All | `CLIENT_URL`, `VITE_API_URL` | Local dev URLs, no account needed |

`.env.example` in both `client/` and `server/` documents every variable name (no real values); real `.env` files stay gitignored.

## Verification

No automated test suite in v1 — verification is running both dev servers and exercising each flow manually/via REST client:

- **Phase 1**: register/login/logout via curl and browser; `/api/auth/me` returns 401 unauthenticated, 200 with cookie; seeded admin logs in with `role:'admin'`.
- **Phase 2**: non-admin POST /api/products → 403; admin creates a product with image upload, confirm image on Cloudinary + URL in Mongo; storefront filters by category, detail page shows correct sizes/stock; zero-stock product shows "out of stock".
- **Phase 3**: cart persists across refresh for logged-in user; COD checkout creates a visible order, decrements stock, clears cart. (Razorpay verification deferred to when that flow is built.)
- **Phase 4**: admin walks an order pending→confirmed→shipped→delivered; statusHistory records each transition; customer's order detail page reflects updates.
- **Phase 5**: invalid form submissions rejected client- and server-side; mobile-width layout check. **Final golden-path**: browse → product detail → add to cart → checkout (both COD and Razorpay test payment) → confirmation shown → admin sees order, updates status → customer sees updated status. This full click-through in the browser is the definition of "done."

## Critical Files

- `server/src/models/Order.js` — ties products, payment, and status together.
- `server/src/controllers/orderController.js` — Razorpay order creation + signature verification + stock decrement; highest-risk correctness area.
- `server/src/middleware/authMiddleware.js` — protect/admin guards used by nearly every route.
- `client/src/context/CartContext.jsx` — integration point between UI, React Query, and `/api/cart`.
- `client/src/lib/api.js` — shared axios instance every frontend call goes through.
