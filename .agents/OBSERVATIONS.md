# accountill — observations

**Scope:** Static read-only review of the server routes and data model, with repository reconnaissance and gap verification. `client/build/` was excluded. No app, build, or test command was run.

## 1. Repository and runtime shape

**Claim:** This is a split MERN project: a React 17 client using `react-scripts` 4.0.3 and an ES-module Express server using Express 4.17.1 and Mongoose 5.12.10. The server package exposes only a `start` script; this review did not run it.

**Evidence:** `README.md:2-3, 43-68`; `client/package.json:25-35, 43-47`; `server/package.json:6-23`. Repository HEAD is `399f2a3` (2023-04-09).

**Confidence:** High.

## 2. Server route surface

**Claim:** The server mounts `/invoices`, `/clients`, `/users`, and `/profiles`; it also defines standalone `/send-pdf`, `/create-pdf`, `/fetch-pdf`, and `/` endpoints. These are API endpoints, not the React client’s page routes.

**Evidence:** `server/index.js:32-35, 53-104`; route tables in `server/routes/{invoices,clients,userRoutes,profile}.js`.

**Confidence:** High.

## 3. Data model

**Claim:** Invoices embed line items, a client snapshot, and payment records rather than referencing separate item/payment/client documents. Ownership is represented by string-array fields (`creator` / `userId`); line-item price, quantity, and discount are strings while invoice totals and payment amounts are numbers.

**Evidence:** `server/models/InvoiceModel.js:3-22`; `server/models/ClientModel.js:4-14`; profile ownership field: `server/models/ProfileModel.js:3-13`.

**Confidence:** High.

## 4. Correction — JWT issuance is not route protection

**Correction to my initial assumption:** I first read the README’s JWT-authentication description as meaning business API requests are JWT-protected. That claim was wrong. The login/signup controller issues JWTs and `middleware/auth.js` contains token verification, but the server mounts the routers without applying that middleware, and the route modules do not attach it. Several handlers also query by caller-supplied query values or document IDs; for example, `GET /clients` returns a global paginated query and `GET /clients/:id`-style ownership checks are absent for mutations. **As written, the server does not enforce authenticated, per-user access on these routes.**

**Evidence:** `server/controllers/user.js:18-37, 46-63`; `server/middleware/auth.js:7-31`; `server/index.js:16-20, 32-35`; `server/routes/clients.js:6-10`; `server/controllers/clients.js:37-47, 67-86, 90-96`; `server/controllers/invoices.js:9-18, 68-100`.

**Confidence:** High for missing middleware/ownership checks in the inspected routes.

## 5. PDF flow has shared-file and error-handling risks

**Claim:** PDF generation writes to the same `server/invoice.pdf` path used as the email attachment and download target. The email send is started before the PDF callback checks its error, and its result is not awaited/handled here; the shared filename also creates a plausible cross-request overwrite risk if requests overlap.

**Evidence:** `server/index.js:53-77` (`/send-pdf`); `server/index.js:87-99` (`/create-pdf`, `/fetch-pdf`).

**Confidence:** High for shared path and ordering; overlap impact is a code-level risk, not a reproduced runtime result.
