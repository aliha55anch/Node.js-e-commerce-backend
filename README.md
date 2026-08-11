# Ecommerce Backend 

## 1. Tech Stack

| Technology  | Purpose                                        |
| ----------- | ---------------------------------------------- |
| Node.js     | JavaScript runtime                             |
| Express.js  | HTTP server / REST API framework               |
| MongoDB     | NoSQL database                                 |
| Mongoose    | ODM for MongoDB (schema + queries)             |
| JWT         | Authentication tokens                          |
| bcryptjs    | Password hashing                               |
| Cloudinary  | Image upload/storage / CDN                   |
| Stripe      | Payment processing (payment intents)           |
| Nodemailer  | Sending emails (password reset)                |
| dotenv      | Loading environment variables                  |
| express-fileupload | File upload middleware                    |
| cookie-parser | Parsing cookies (for JWT token)             |

---

## 2. Folder Structure

```
backend/
├── app.js                     # Express app setup + middleware + route mounting
├── server.js                  # Entry point: starts the server, connects DB
├── config/
│   ├── database.js            # MongoDB connection logic
│   └── config.env             # Environment variables (DB URI, JWT secret, Stripe keys, etc.)
├── models/
│   ├── userModel.js           # User schema (auth, password hashing, JWT, reset tokens)
│   ├── productModel.js        # Product schema (name, price, images, reviews)
│   └── orderModel.js          # Order schema (shipping, items, payment, prices)
├── controllers/
│   ├── userController.js      # Register, login, logout, password reset, profile, admin user ops
│   ├── productController.js   # CRUD products, search/filter/pagination, reviews
│   ├── orderController.js     # Create/read/update/delete orders, stock management
│   └── paymentController.js   # Stripe payment intents + API key
├── routes/
│   ├── userRoute.js           # /api/v1/... user endpoints
│   ├── productRoute.js        # /api/v1/... product endpoints
│   ├── orderRoute.js          # /api/v1/... order endpoints
│   └── paymentRoute.js        # /api/v1/... payment endpoints
├── middleware/
│   ├── error.js               # Global error handler (Mongoose/JWT errors normalized)
│   ├── catchAsyncErrors.js    # Wraps async controllers, forwards errors to next()
│   └── auth.js                # isAuthenticatedUser + authorizeRoles
└── utils/
    ├── errorhander.js         # Custom ErrorHandler class (message + statusCode)
    ├── jwtToken.js            # Creates JWT + stores it in an httpOnly cookie
    ├── apifeatures.js         # Search, filter, pagination helpers for product queries
    └── sendEmail.js           # Nodemailer transporter for sending emails
```

---

## 3. How the Server Starts (`server.js`)

`server.js` is the entry point. It runs in this order:

1. **Loads `app.js`** — the configured Express application.
2. **Registers an `uncaughtException` handler** — if any synchronous code throws an uncaught error, it logs the message and shuts the server down with `process.exit(1)`.
3. **Loads environment variables** from `backend/config/config.env` (only when `NODE_ENV` is not `"PRODUCTION"`, because in production env vars come from the hosting platform).
4. **Connects to MongoDB** by calling `connectDatabase()`.
5. **Configures Cloudinary** with `CLOUDINARY_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` from env — used later for image uploads.
6. **Starts listening** on `process.env.PORT`.
7. **Registers an `unhandledRejection` handler** — if any promise rejects without a catch, it logs the error, closes the server gracefully, then exits.

> **Why two shutdown handlers?** `uncaughtException` = sync errors, `unhandledRejection` = async errors. Both force a clean exit so the process doesn't stay alive in a broken state.

---

## 4. The Express App (`app.js`)

`app.js` builds the Express application:

1. **Imports Express** and creates `app`.
2. **Registers global middleware** (runs on every request):
   - `express.json()` — parses JSON request bodies.
   - `cookieParser()` — parses cookies so `req.cookies.token` is available.
   - `bodyParser.urlencoded({ extended: true })` — parses form-encoded bodies.
   - `express-fileupload()` — makes uploaded files available as `req.files`.
3. **Mounts the 4 routers** under the `/api/v1` prefix:
   - `app.use("/api/v1", product)`
   - `app.use("/api/v1", user)`
   - `app.use("/api/v1", order)`
   - `app.use("/api/v1", payment)`
   - So a route defined as `/products` in `productRoute.js` becomes `/api/v1/products`.
4. **Serves the built React frontend**:
   - `express.static` serves static files from `../frontend/build`.
   - `app.get("*")` sends `index.html` for any unmatched route — this is how the React app handles its own client-side routing in production.
5. **Registers the global error middleware** as the very last middleware — Express only calls it when an error is passed to `next()`.

---

## 5. Request Flow — How a Request is Handled

A typical authenticated request (e.g. `GET /api/v1/me`) flows like this:

```
Browser sends request
      │
      ▼
Global middleware (JSON, cookies, urlencoded, fileupload)
      │
      ▼
Route matching: /api/v1/me  → userRoute.js
      │
      ▼
Middleware chain: isAuthenticatedUser → getUserDetails
      │
      ▼
Controller does the DB work (User.findById(req.user.id))
      │
      ▼
Responds with JSON { success: true, user }
```

**If anything throws an error** (e.g. invalid token):

```
Controller calls next(new ErrorHandler("Please Login...", 401))
      │
      ▼
catchAsyncErrors catches the rejected promise → next(err)
      │
      ▼
Global error middleware (middleware/error.js) formats the response
      │
      ▼
Resolves with { success: false, message: "Please Login..." } + status 401
```

---

## 6. Configuration (`config/`)

### `config/database.js`
Exports `connectDatabase()` which calls `mongoose.connect(process.env.DB_URI, ...)` and logs the MongoDB host once connected. It uses legacy options (`useNewUrlParser`, `useUnifiedTopology`, `useCreateIndex`) to suppress deprecation warnings.

### `config/config.env`
Holds all secrets and settings. **This file is NOT in the repo** (it is git-ignored). You must create it. Example content:

```env
PORT=4000
DB_URI=mongodb://localhost:27017/ecommerce
NODE_ENV=DEVELOPMENT

JWT_SECRET=your_jwt_secret
JWT_EXPIRE=5d
COOKIE_EXPIRE=5

CLOUDINARY_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

SMPT_HOST=smtp.gmail.com
SMPT_PORT=465
SMPT_SERVICE=gmail
SMPT_MAIL=you@gmail.com
SMPT_PASSWORD=your_app_password

STRIPE_SECRET_KEY=sk_test_...
STRIPE_API_KEY=pk_test_...
```

---

## 7. Models (`models/`)

### `userModel.js`
Schema fields: `name`, `email` (unique), `password` (hashed, `select: false` so it is never returned by default), `avatar` (`public_id` + `url` from Cloudinary), `role` (`user` by default), `createdAt`, `resetPasswordToken`, `resetPasswordExpire`.

Key logic:
- **`pre("save")` hook** — hashes the password with `bcrypt.hash(password, 10)` before saving, but only if the password was modified.
- **`getJWTToken()`** — signs a JWT containing the user `id`, valid for `JWT_EXPIRE`.
- **`comparePassword(password)`** — bcrypt-compares a plain password with the stored hash.
- **`getResetPasswordToken()`** — generates a random 20-byte hex token, stores its SHA-256 hash on the user, sets expiry to 15 minutes, and returns the **plain** token (the un-hashed one is what gets emailed).

### `productModel.js`
Schema fields: `name`, `description`, `price`, `ratings` (average rating, default 0), `images[]` (array of `{public_id, url}`), `category`, `Stock`, `numOfReviews`, `reviews[]` (each: `user`, `name`, `rating`, `comment`), `user` (admin who created it), `createdAt`.

### `orderModel.js`
Schema fields:
- `shippingInfo` — `address`, `city`, `state`, `country`, `pinCode`, `phoneNo`.
- `orderItems[]` — snapshot of `name`, `price`, `quantity`, `image`, and the `product` reference.
- `user` — reference to the buyer.
- `paymentInfo` — `id` + `status` (from Stripe).
- `paidAt`, `itemsPrice`, `taxPrice`, `shippingPrice`, `totalPrice`.
- `orderStatus` — default `"Processing"`, changes to `"Shipped"` / `"Delivered"`.
- `deliveredAt`, `createdAt`.

---

## 8. Middleware (`middleware/`)

### `catchAsyncErrors.js`
A wrapper that takes an async controller function and returns `(req, res, next) => Promise.resolve(theFunc(req, res, next)).catch(next)`. This means **any rejected promise automatically calls `next(error)`** — you never write try/catch in controllers.

### `error.js` (global error handler)
Has 4 parameters (`err, req, res, next`) — Express recognizes it as error middleware. It:
1. Defaults `statusCode` to 500 and `message` to `"Internal Server Error"`.
2. Normalizes common errors:
   - **`CastError`** → invalid MongoDB `_id` → "Resource not found. Invalid: <path>" (400).
   - **Duplicate key (code 11000)** → "Duplicate <field> Entered" (400).
   - **`JsonWebTokenError`** → invalid token (400).
   - **`TokenExpiredError`** → expired token (400).
3. Responds with `{ success: false, message }`.

### `auth.js`
- **`isAuthenticatedUser`** — reads the `token` cookie. If missing → 401 "Please Login to access this resource". Otherwise it verifies the token with `jwt.verify(token, JWT_SECRET)`, loads the user with `User.findById(decodedData.id)`, and attaches them to `req.user`. The `next()` then runs the controller.
- **`authorizeRoles(...roles)`** — returns a middleware that checks `req.user.role` is one of the allowed roles (e.g. `"admin"`). If not → 403 "Role is not allowed to access this resource".

---

## 9. Utils (`utils/`)

### `errorhander.js`
Defines `class ErrorHandler extends Error` with `statusCode`. Every controller error is created with `new ErrorHandler(message, statusCode)` so the global handler knows what HTTP status to return.

### `jwtToken.js`
`sendToken(user, statusCode, res)`:
1. Calls `user.getJWTToken()` to create the token.
2. Builds a cookie that expires in `COOKIE_EXPIRE` days, marked `httpOnly: true` (JavaScript on the client cannot read it — safer against XSS).
3. Sends the response with `success`, `user`, and `token` while setting the cookie.

### `apifeatures.js`
`class ApiFeatures(query, queryStr)` — builds MongoDB queries from query-string params. Used for `GET /products`.
- **`search()`** — if `?keyword=...`, adds a case-insensitive regex on the `name` field.
- **`filter()`** — copies the query string, removes `keyword`/`page`/`limit`, then converts `gt|gte|lt|lte` into MongoDB operators (`$gt`, `$gte`, `$lt`, `$lte`). This powers `?price[gte]=1000&category=Laptops`.
- **`pagination(resultPerPage)`** — computes `skip` from the current page and applies `.limit().skip()`.

### `sendEmail.js`
Creates a Nodemailer transporter from `SMPT_*` env vars and sends an email with `subject` and `message`. Used for password recovery.

---

## 10. Controllers (`controllers/`)

### `userController.js`
| Function | What it does |
| --- | --- |
| `registerUser` | Uploads the avatar to Cloudinary, creates the user, sends JWT cookie (201). |
| `loginUser` | Finds user by email (with `+password`), compares hashes, sends JWT cookie (200). |
| `logout` | Expires the cookie immediately, responds "Logged Out". |
| `forgotPassword` | Finds user, generates reset token, emails the reset URL, stores the hashed token + 15 min expiry. On email failure it clears the token fields and returns 500. |
| `resetPassword` | Hashes the token from the URL, finds the user by token + expiry, checks `password === confirmPassword`, saves new password (auto-hashed), sends JWT. |
| `getUserDetails` | Returns the logged-in user's own profile. |
| `updatePassword` | Verifies old password, sets new password, sends fresh JWT. |
| `updateProfile` | Updates name/email; if a new avatar is sent, deletes the old one from Cloudinary and uploads the new one. |
| `getAllUser` (admin) | Lists all users. |
| `getSingleUser` (admin) | Returns one user by id. |
| `updateUserRole` (admin) | Updates name/email/role of a user. |
| `deleteUser` (admin) | Deletes the user's Cloudinary avatar then removes the user. |

### `productController.js`
| Function | What it does |
| --- | --- |
| `createProduct` (admin) | Accepts base64 image strings, uploads each to Cloudinary, stores `{public_id, url}` links, attaches the admin's `user` id, creates the product. |
| `getAllProducts` | Uses `ApiFeatures` to search + filter, counts total/filtered products, applies pagination (8 per page), returns everything the frontend needs for the product list page. |
| `getAdminProducts` (admin) | Returns all products (no pagination) for the admin dashboard. |
| `getProductDetails` | Returns one product by `:id` or 404. |
| `updateProduct` (admin) | If new images are sent, destroys old Cloudinary images first, uploads new ones, then updates the product. |
| `deleteProduct` (admin) | Destroys all Cloudinary images, then removes the product. |
| `createProductReview` | Adds a review, or updates the user's existing review, then recomputes `ratings` (average) and `numOfReviews`. |
| `getProductReviews` | Returns `product.reviews` by `?id=` query. |
| `deleteReview` | Filters out the review, recomputes average rating and count, updates the product. |

### `orderController.js`
| Function | What it does |
| --- | --- |
| `newOrder` | Creates an order from the request body, sets `paidAt` and the current user. |
| `getSingleOrder` | Returns an order by `:id`, `populate("user", "name email")` so it includes the buyer's name/email. |
| `myOrders` | Returns all orders where `user == req.user._id` (the logged-in customer's orders). |
| `getAllOrders` (admin) | Returns all orders plus the `totalAmount` (sum of all `totalPrice`). |
| `updateOrder` (admin) | Sets the new status. When status becomes `"Shipped"` it calls `updateStock` per item (deducts quantity from `product.Stock`). When `"Delivered"` it records `deliveredAt`. Refuses to change an already-delivered order. |
| `deleteOrder` (admin) | Removes an order by id. |

### `paymentController.js`
| Function | What it does |
| --- | --- |
| `processPayment` | Creates a Stripe **payment intent** for `req.body.amount` in INR, returns the `client_secret` (the frontend uses it to confirm the card payment). |
| `sendStripeApiKey` | Returns the publishable Stripe API key so the frontend can initialize Stripe. |

---

## 11. Routes (`routes/`) — Full API Reference

All routes are prefixed with `/api/v1`.

### User routes (`userRoute.js`)
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| POST | `/register` | Public | Create account |
| POST | `/login` | Public | Login |
| GET | `/logout` | Public | Logout (clears cookie) |
| POST | `/password/forgot` | Public | Request password reset email |
| PUT | `/password/reset/:token` | Public | Set new password |
| GET | `/me` | User | Get own profile |
| PUT | `/password/update` | User | Change password |
| PUT | `/me/update` | User | Update name/email/avatar |
| GET | `/admin/users` | Admin | List all users |
| GET | `/admin/user/:id` | Admin | Get one user |
| PUT | `/admin/user/:id` | Admin | Update user role |
| DELETE | `/admin/user/:id` | Admin | Delete user |

### Product routes (`productRoute.js`)
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| GET | `/products` | Public | All products (search/filter/pagination) |
| GET | `/product/:id` | Public | Product details |
| GET | `/admin/products` | Admin | All products (dashboard) |
| POST | `/admin/product/new` | Admin | Create product |
| PUT | `/admin/product/:id` | Admin | Update product |
| DELETE | `/admin/product/:id` | Admin | Delete product |
| PUT | `/review` | User | Add/update review |
| GET | `/reviews` | Public | Get reviews `?id=productId` |
| DELETE | `/reviews` | User | Delete a review `?productId=..&id=..` |

### Order routes (`orderRoute.js`)
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| POST | `/order/new` | User | Place an order |
| GET | `/order/:id` | User | Get one order |
| GET | `/orders/me` | User | My orders |
| GET | `/admin/orders` | Admin | All orders + total amount |
| PUT | `/admin/order/:id` | Admin | Update order status |
| DELETE | `/admin/order/:id` | Admin | Delete order |

### Payment routes (`paymentRoute.js`)
| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| POST | `/payment/process` | User | Create Stripe payment intent |
| GET | `/stripeapikey` | User | Get Stripe publishable key |

---

## 12. Security Notes

- Passwords are hashed with bcrypt (10 salt rounds).
- The JWT lives in an **httpOnly cookie**, so client-side JS can't steal it.
- Password-reset tokens are stored **hashed** in the DB and expire after 15 minutes.
- Admin-only routes are protected with `isAuthenticatedUser` + `authorizeRoles("admin")`.
- Global error handler normalizes known errors (invalid ObjectId, duplicate keys, bad/expired JWTs) into clean JSON responses.

---

## 13. How to Run

1. Install dependencies (from the backend root, or repo root):
   ```bash
   npm install
   ```
2. Create `backend/config/config.env` with the variables from Section 6.
3. Make sure MongoDB is running locally (or point `DB_URI` at a cluster).
4. Start the server:
   ```bash
   npm run dev      # development (uses nodemon if configured)
   # or
   node backend/server.js
   ```
5. The API is now available at `http://localhost:4000/api/v1/...`

> The `package.json` lives at the repo root (this folder only contains the backend source). `npm run dev` is typically defined there as `nodemon backend/server.js`.
