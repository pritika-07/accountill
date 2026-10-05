# Tech Stack Overview

## 1. Tech Stack — Languages, Frameworks, Database & Major Libraries

### Languages
- **JavaScript (ES6+ with ES Modules)** — used across both client and server.
  - Evidence: `"type": "module"` in `server/package.json:7`.
  - Client uses JSX via React (`client/src/App.js:4`).

### Frontend Framework
| Library | Version | Evidence |
|---|---|---|
| React | `^17.0.2` | `client/package.json:25` |
| React DOM | `^17.0.2` | `client/package.json:27` |
| React Router DOM | `^5.2.0` | `client/package.json:33` |
| Redux | `^4.1.0` | `client/package.json:38` |
| React-Redux | `^7.2.4` | `client/package.json:32` |
| Redux Thunk | `^2.3.0` | `client/package.json:39` |
| Material UI Core | `^4.11.4` | `client/package.json:10` |
| Material UI Icons | `^4.11.2` | `client/package.json:11` |
| Material UI Lab | `^4.0.0-alpha.58` | `client/package.json:12` |
| Material UI Pickers | `^3.3.10` | `client/package.json:13` |
| Axios | `^0.21.1` | `client/package.json:19` |
| ApexCharts | `^3.28.1` | `client/package.json:18` |
| React ApexCharts | `^1.3.9` | `client/package.json:26` |
| Recharts | `^2.0.9` | `client/package.json:37` |
| @react-oauth/google | `^0.9.0` | `client/package.json:14` |
| React Dropzone | `^11.3.4` | `client/package.json:28` |
| Moment.js | `^2.29.1` | `client/package.json:24` |
| Lodash | `^4.17.21` | `client/package.json:23` |
| uuid | `^8.3.2` | `client/package.json:40` |
| react-simple-snackbar | `^1.1.11` | `client/package.json:35` |
| file-saver | `^2.0.5` | `client/package.json:21` |
| jwt-decode | `^3.1.2` | `client/package.json:22` |
| react-scripts | `4.0.3` | `client/package.json:34` |

### Backend Framework
| Library | Version | Evidence |
|---|---|---|
| Express | `^4.17.1` | `server/package.json:18` |
| Mongoose | `^5.12.10` | `server/package.json:22` |
| bcryptjs | `^2.4.3` | `server/package.json:15` |
| jsonwebtoken | `^8.5.1` | `server/package.json:20` |
| Nodemailer | `^6.6.3` | `server/package.json:23` |
| html-pdf | `^3.0.1` | `server/package.json:19` |
| cors | `^2.8.5` | `server/package.json:16` |
| dotenv | `^8.5.0` | `server/package.json:17` |
| Moment.js | `^2.29.1` | `server/package.json:21` |
| nodemon (dev) | `^2.0.7` | `server/package.json:26` |

### Database
- **MongoDB** via **MongoDB Atlas** (cloud-hosted).
  - Evidence: Connection string loaded from `process.env.DB_URL` at `server/index.js:106–109`.
  - Mongoose driver at `server/package.json:22`.

---

## 2. How to Run Locally

### Prerequisites
- Node.js (compatible with react-scripts 4.0.3, so Node 12–14 recommended)
- MongoDB Atlas account or local MongoDB instance

### Client Setup
```bash
cd client
npm install
npm start        # Starts dev server on http://localhost:3000
```
**Required environment variables** (in `client/.env`):
- `REACT_APP_GOOGLE_CLIENT_ID`
- `REACT_APP_API` (e.g., `http://localhost:5000`)
- `REACT_APP_URL` (e.g., `http://localhost:3000`)

Evidence: `README.md:80–85`

### Server Setup
```bash
cd server
npm install
npm start        # Runs `node index.js` on PORT (default 5000)
```
**Required environment variables** (in `server/.env`):
- `DB_URL` — MongoDB connection string
- `PORT` — Server port (defaults to 5000)
- `SECRET` — JWT signing secret
- `SMTP_HOST` — Email host
- `SMTP_PORT` — Email port
- `SMTP_USER` — Email username
- `SMTP_PASS` — Email password

Evidence: `README.md:100–113`, `server/index.js:106–107`, `server/controllers/user.js:8–12`

### Docker Setup
```bash
docker-compose -f docker-compose.prod.yml build
docker-compose -f docker-compose.prod.yml up
```
Evidence: `docker-compose.prod.yml`, `README.md:133–164`

---

## 3. Folder Map

### Top-Level Directories
| Folder / File | Purpose |
|---|---|
| `client/` | React SPA frontend — all UI components, Redux state, API calls |
| `server/` | Express.js backend — REST API, authentication, PDF generation, email |
| `.github/` | GitHub-specific configuration (CI/CD, templates) |
| `docker-compose.prod.yml` | Docker Compose config for production deployment |
| `README.md` | Project documentation and setup guide |
| `LICENSE.md` | MIT License |
| `prompts.md` | Reverse engineering prompt templates |

### 10 Most Important Files
| # | File | Purpose |
|---|---|---|
| 1 | `server/index.js` | Express app entry point — mounts all routes, configures CORS, connects to MongoDB, handles PDF generation and email sending |
| 2 | `client/src/App.js` | React app root — defines all client-side routes via React Router |
| 3 | `server/models/InvoiceModel.js` | Mongoose schema for invoices — the core data entity with items, payments, client info |
| 4 | `server/controllers/user.js` | Handles signup, signin, password forgot/reset with bcrypt hashing and JWT creation |
| 5 | `server/middleware/auth.js` | JWT authentication middleware — differentiates custom JWT and Google tokens |
| 6 | `server/controllers/invoices.js` | CRUD operations for invoices (create, read, update, delete) |
| 7 | `client/src/api/index.js` | Axios HTTP client — all API endpoint definitions with JWT interceptor |
| 8 | `server/documents/index.js` | HTML template for generating PDF invoices via html-pdf |
| 9 | `client/src/actions/auth.js` | Redux thunks for signin, signup, forgot password, reset password |
| 10 | `server/models/userModel.js` | Mongoose schema for user accounts — email/password/reset tokens |

---

## 4. Odd Files

| File | Why it looks odd |
|---|---|
| `server/invoice.pdf` (29 KB) | **Build artifact committed to repo.** This is a generated PDF that `html-pdf` writes to disk at `server/index.js:57`. It should be in `.gitignore`. |
| `client/src/store.js` | **Dead code / scratch file.** Contains array data for "GPS", "mistnet", "camera" lending — has nothing to do with Redux or invoicing. It is never imported by the actual Redux setup. Evidence: `client/src/store.js:6–35` |
| `client/src/clients.json` (6 KB) | **Static JSON fixture.** Appears to be sample/mock client data. Not imported by any component found in the codebase. |
| `client/src/currencies.json` (67 KB) | **Large static dataset.** Contains a comprehensive list of world currencies. Reasonable for a currency selector, but at 67 KB it is unusually large for a frontend bundle. |
| `server/documents/invoice.js` (3.8 KB) | **Possibly unused template.** The import in `server/index.js:22` is commented out (`// import invoiceTemplate`). The active template is `server/documents/index.js`. |
| `server/Procfile` | **Heroku deployment artifact.** Contains `web: npm start`. Only relevant if deploying to Heroku. |
