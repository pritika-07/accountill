# Verified Claims

## Overview
This document re-verifies every major claim made during the analysis of the Accountill codebase. Each claim is tagged as **Confirmed** (re-read the exact line), **Likely** (strong signs but no single line proves it), or **Guess** (no direct evidence).

---

## Tech Stack Claims

- The client uses **React 17.0.2**
  Evidence: `client/package.json:25` — `"react": "^17.0.2"` [Confirmed]

- The client uses **Redux 4.1.0** with **Redux Thunk 2.3.0** for state management
  Evidence: `client/package.json:38` — `"redux": "^4.1.0"`, `client/package.json:39` — `"redux-thunk": "^2.3.0"` [Confirmed]

- The client uses **Material UI 4.11.4** for UI components
  Evidence: `client/package.json:10` — `"@material-ui/core": "^4.11.4"` [Confirmed]

- The client uses **React Router DOM 5.2.0** for routing
  Evidence: `client/package.json:33` — `"react-router-dom": "^5.2.0"` [Confirmed]

- The client uses **Axios 0.21.1** for HTTP requests
  Evidence: `client/package.json:19` — `"axios": "^0.21.1"` [Confirmed]

- The server uses **Express 4.17.1**
  Evidence: `server/package.json:18` — `"express": "^4.17.1"` [Confirmed]

- The server uses **Mongoose 5.12.10** for MongoDB ODM
  Evidence: `server/package.json:22` — `"mongoose": "^5.12.10"` [Confirmed]

- The server uses **jsonwebtoken 8.5.1** for JWT handling
  Evidence: `server/package.json:20` — `"jsonwebtoken": "^8.5.1"` [Confirmed]

- The server uses **bcryptjs 2.4.3** for password hashing
  Evidence: `server/package.json:15` — `"bcryptjs": "^2.4.3"` [Confirmed]

- The server uses **Nodemailer 6.6.3** for email
  Evidence: `server/package.json:23` — `"nodemailer": "^6.6.3"` [Confirmed]

- The server uses **html-pdf 3.0.1** for PDF generation
  Evidence: `server/package.json:19` — `"html-pdf": "^3.0.1"` [Confirmed]

- The database is **MongoDB** via **MongoDB Atlas**
  Evidence: `server/index.js:109` — `mongoose.connect(DB_URL, ...)` and `README.md:68` — "MongoDB (MongoDB Atlas)" [Confirmed]

- The server uses **ES Modules** (import/export syntax)
  Evidence: `server/package.json:7` — `"type": "module"` [Confirmed]

---

## Architecture Claims

- The client uses `localStorage` to persist JWT tokens
  Evidence: `client/src/App.js:23` — `JSON.parse(localStorage.getItem('profile'))` and `client/src/api/index.js:7–8` — reads token from localStorage [Confirmed]

- Axios interceptor attaches Bearer token to all requests
  Evidence: `client/src/api/index.js:6–12` — `req.headers.authorization = 'Bearer ${...token}'` [Confirmed]

- Express body parser is set to 30MB limit
  Evidence: `server/index.js:28` — `express.json({ limit: "30mb", extended: true })` [Confirmed]

- CORS is enabled globally
  Evidence: `server/index.js:30` — `app.use((cors()))` [Confirmed]

- 4 route files are mounted: invoices, clients, users, profiles
  Evidence: `server/index.js:32–35` — all four `app.use()` calls confirmed [Confirmed]

- PDF operations write to a single shared `invoice.pdf` file on disk
  Evidence: `server/index.js:57` — `.toFile('invoice.pdf', ...)`, `server/index.js:69` — `path: '${__dirname}/invoice.pdf'` [Confirmed]

- The NavBar is shown only when a user is logged in
  Evidence: `client/src/App.js:29` — `{user && <NavBar />}` [Confirmed]

- Password reset link is hardcoded to `https://accountill.com`
  Evidence: `server/controllers/user.js:122` — `href="https://accountill.com/reset/${token}"` [Confirmed]

---

## Data Model Claims

- The User model has `name`, `email` (unique), `password`, `resetToken`, `expireToken`
  Evidence: `server/models/userModel.js:3–9` [Confirmed]

- Profile stores `userId` as `[String]` not as Mongoose ObjectId reference
  Evidence: `server/models/ProfileModel.js:12` — `userId: [String]` [Confirmed]

- Invoice embeds client details as a sub-object (`{ name, email, phone, address }`)
  Evidence: `server/models/InvoiceModel.js:17` [Confirmed]

- Invoice embeds payment records as an array
  Evidence: `server/models/InvoiceModel.js:18` [Confirmed]

- Invoice items store `unitPrice`, `quantity`, `discount` as `String` not `Number`
  Evidence: `server/models/InvoiceModel.js:6` — `unitPrice: String, quantity: String, discount: String` [Confirmed]

- `createdAt` default uses `new Date()` (evaluated once at schema load)
  Evidence: `server/models/InvoiceModel.js:21` — `default: new Date()`, `server/models/ClientModel.js:12` — same [Confirmed]

- `User.bio` is passed during signup but not in the schema
  Evidence: `server/controllers/user.js:47` destructures `bio`, but `server/models/userModel.js:3–9` has no `bio` field [Confirmed]

---

## Security Claims

- **Auth middleware is never applied to any route file**
  Evidence: Searched all route files — `server/routes/invoices.js`, `server/routes/clients.js`, `server/routes/userRoutes.js`, `server/routes/profile.js` — none import `auth.js` [Confirmed]

- Auth middleware distinguishes JWT vs Google tokens by `token.length < 500`
  Evidence: `server/middleware/auth.js:10` — `const isCustomAuth = token.length < 500` [Confirmed]

- Auth middleware fails silently on error (no response sent)
  Evidence: `server/middleware/auth.js:29–31` — `catch (error) { console.log(error) }` — no `res.status(401)` or `next()` [Confirmed]

- No ownership check on update/delete operations
  Evidence: `server/controllers/invoices.js:85` checks ObjectId validity but not ownership. `server/controllers/clients.js:71` — same. `server/controllers/profile.js:105` — same. [Confirmed]

- TLS certificate verification is disabled in Nodemailer config
  Evidence: `server/index.js:46` — `rejectUnauthorized: false` [Confirmed]

- `/send-pdf` endpoint has no authentication
  Evidence: `server/index.js:53` — `app.post('/send-pdf', (req, res) =>` — no middleware parameter [Confirmed]

- Password is hashed with bcrypt using 12 salt rounds
  Evidence: `server/controllers/user.js:57` — `bcrypt.hash(password, 12)` [Confirmed]

- JWT expires in 1 hour
  Evidence: `server/controllers/user.js:34` — `jwt.sign({...}, SECRET, { expiresIn: "1h" })` [Confirmed]

- Reset token expires in 1 hour
  Evidence: `server/controllers/user.js:114` — `user.expireToken = Date.now() + 3600000` [Confirmed]

---

## Feature Claims

- Signup auto-creates a blank profile from the client side
  Evidence: `client/src/actions/auth.js:30` — `api.createProfile({name, email, userId, ...})` [Confirmed]

- Invoice number is generated randomly: `Math.floor(Math.random() * 100000)`
  Evidence: `client/src/initialState.js:13` [Confirmed]

- Client pagination uses LIMIT of 8
  Evidence: `server/controllers/clients.js:41` — `const LIMIT = 8` [Confirmed]

- `getInvoice` action also fetches the user's business profile
  Evidence: `client/src/actions/invoiceActions.js:33` — `api.fetchProfilesByUser(...)` [Confirmed]

- After signup, redirect uses `window.location.href` (full page reload, not React Router)
  Evidence: `client/src/actions/auth.js:15` — `window.location.href="/dashboard"` [Confirmed]

- Regex search on profiles uses raw user input in `RegExp` constructor
  Evidence: `server/controllers/profile.js:89–90` — `new RegExp(searchQuery, "i")` [Confirmed]

---

## Odd Files Claims

- `server/invoice.pdf` is a generated artifact committed to the repo
  Evidence: File exists at `server/invoice.pdf` (29,308 bytes) [Confirmed]

- `client/src/store.js` contains unrelated equipment/lending data
  Evidence: `client/src/store.js:6–13` — arrays of GPS, mistnet, camera objects [Confirmed]

- `server/documents/invoice.js` is unused — its import is commented out
  Evidence: `server/index.js:22` — `// import invoiceTemplate from './documents/invoice.js'` [Confirmed]

- `server/Procfile` contains `web: npm start` (Heroku artifact)
  Evidence: `server/Procfile` exists (18 bytes) [Confirmed]

---

## README Claims

- README says "React simple Snackbar" — package exists in dependencies
  Evidence: `client/package.json:35` — `"react-simple-snackbar": "^1.1.11"` [Confirmed]

- README says "React-google-login" but actual package is `@react-oauth/google`
  Evidence: `README.md:56` says "React-google-login", but `client/package.json:14` has `"@react-oauth/google": "^0.9.0"` [Confirmed — docs drift]

- README claims "Cloudinary to allow users to upload their business logo"
  Evidence: `README.md:54`. No Cloudinary SDK in any `package.json`. Logo field is a String URL in `ProfileModel.js:10`. Upload mechanism is likely client-side via react-dropzone. [Likely — Cloudinary used for hosting, not via SDK]

---

## Corrections

| Original Claim | Correction | Reason |
|---|---|---|
| "Automatic status change when payment is added" (from README analysis) | **Dropped** | No server-side logic found for automatic status updates. The `status` field exists but is never automatically modified by the backend. |
| "Dashboard uses ApexCharts and Recharts" | **Likely** | Both libraries are in `package.json` but without inspecting the Dashboard component internals, cannot confirm which is actually rendered. Both are listed as dependencies. |
| "Cloudinary SDK used for uploads" | **Corrected** | No Cloudinary SDK found in dependencies. Cloudinary is likely used as a hosting URL target, with uploads handled via react-dropzone on the client side. |
