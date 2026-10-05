# Feature Trace: User Signs Up and Creates Their First Invoice

This document traces two key features end-to-end through the entire codebase, following every function call from the browser to the database and back.

---

## Trace 1: User Registration (Signup)

### Sequence Diagram
```mermaid
sequenceDiagram
    actor U as User
    participant UI as React Login Component
    participant RD as Redux (auth.js action)
    participant AX as Axios API Layer
    participant EX as Express Server
    participant CT as user.js Controller
    participant DB as MongoDB

    U->>UI: Fills firstName, lastName, email, password, confirmPassword
    U->>UI: Clicks "Sign Up"
    UI->>RD: dispatch(signup(formData, openSnackbar, setLoading))
    RD->>AX: api.signUp(formData)
    AX->>EX: POST /users/signup (JSON body)
    EX->>CT: signup(req, res)

    Note over CT: Check 1: Duplicate email
    CT->>DB: User.findOne({ email })
    DB-->>CT: null (no existing user)

    Note over CT: Check 2: Password confirmation
    CT->>CT: password !== confirmPassword?

    Note over CT: Hash password
    CT->>CT: bcrypt.hash(password, 12)
    CT->>DB: User.create({ email, password: hashedPassword, name })
    DB-->>CT: User document created

    Note over CT: Generate JWT
    CT->>CT: jwt.sign({ email, id }, SECRET, { expiresIn: "1h" })

    CT-->>AX: 200 { result: User, userProfile: null, token: JWT }
    AX-->>RD: data

    Note over RD: Store auth in localStorage
    RD->>RD: dispatch({ type: AUTH, data })

    Note over RD: Auto-create profile
    RD->>AX: api.createProfile({ name, email, userId, ... })
    AX->>EX: POST /profiles (JSON body)
    EX->>DB: Profile.save()
    DB-->>EX: Profile created
    EX-->>AX: 201 Profile
    AX-->>RD: profile data
    RD->>RD: dispatch({ type: CREATE_PROFILE, payload })

    RD->>UI: window.location.href = "/dashboard"
    UI-->>U: Redirect to Dashboard
```

### Step-by-Step Walkthrough

| # | Step | What Happens | Data Read/Written | Evidence |
|---|---|---|---|---|
| 1 | User submits signup form | `formData` = `{ firstName, lastName, email, password, confirmPassword }` | — | `client/src/actions/auth.js:24` |
| 2 | Redux action dispatched | `signup` thunk calls `api.signUp(formData)` | — | `client/src/actions/auth.js:28` |
| 3 | Axios sends POST | `POST /users/signup` with JSON body. Bearer token is **not** attached (no profile in localStorage yet). | — | `client/src/api/index.js:30` |
| 4 | **Check: Duplicate email** | `User.findOne({ email })` — if user exists, returns `400: "User already exist"` | Reads: `users` collection | `server/controllers/user.js:50,53` |
| 5 | **Check: Password match** | `password !== confirmPassword` — if mismatch, returns `400: "Password don't match"` | — | `server/controllers/user.js:55` |
| 6 | Hash password | `bcrypt.hash(password, 12)` — 12 salt rounds | — | `server/controllers/user.js:57` |
| 7 | Create user | `User.create({ email, password: hashedPassword, name: '${firstName} ${lastName}' })` | Writes: `users` collection | `server/controllers/user.js:59` |
| 8 | Generate token | `jwt.sign({ email, id }, SECRET, { expiresIn: "1h" })` | — | `server/controllers/user.js:61` |
| 9 | Return response | `200 { result, userProfile: (undefined on signup), token }` | — | `server/controllers/user.js:63` |
| 10 | Client stores auth | Redux `AUTH` action stores data. LocalStorage set via reducer. | Writes: `localStorage['profile']` | `client/src/actions/auth.js:29` |
| 11 | Auto-create profile | Client calls `api.createProfile(...)` with blank business fields | Writes: `profiles` collection | `client/src/actions/auth.js:30` |
| 12 | Redirect | `window.location.href = "/dashboard"` (full page reload) | — | `client/src/actions/auth.js:32` |

### Failure Scenarios
| Failure | Error Returned | Database State | Evidence |
|---|---|---|---|
| Duplicate email | `400: "User already exist"` | No change | `controllers/user.js:53` |
| Password mismatch | `400: "Password don't match"` | No change | `controllers/user.js:55` |
| Server error (DB down) | `500: "Something went wrong"` | No change | `controllers/user.js:66` |
| Profile creation fails after user created | User exists in DB but has no Profile | User is orphaned without a profile — dashboard may error | `client/src/actions/auth.js:30` (no rollback) |

---

## Trace 2: Creating and Sending an Invoice

### Sequence Diagram
```mermaid
sequenceDiagram
    actor U as Business Owner
    participant INV as Invoice Component
    participant RD as Redux (invoiceActions.js)
    participant AX as Axios API Layer
    participant EX as Express Server
    participant IC as invoices.js Controller
    participant DB as MongoDB
    participant PDF as html-pdf Engine
    participant TPL as documents/index.js Template
    participant SMTP as Email Provider

    rect rgb(230, 245, 255)
    Note over U, DB: Phase 1: Create Invoice
    U->>INV: Fills items, client, amounts, due date
    U->>INV: Clicks "Save Invoice"
    INV->>RD: dispatch(createInvoice(invoice, history))
    RD->>RD: dispatch({ type: START_LOADING })
    RD->>AX: api.addInvoice(invoice)
    AX->>EX: POST /invoices + Bearer Token
    EX->>IC: createInvoice(req, res)
    IC->>IC: const newInvoice = new InvoiceModel(req.body)
    IC->>DB: newInvoice.save()
    DB-->>IC: Saved document with _id
    IC-->>AX: 201 { newInvoice }
    AX-->>RD: data
    RD->>RD: dispatch({ type: ADD_NEW, payload: data })
    RD->>INV: history.push("/invoice/${data._id}")
    RD->>RD: dispatch({ type: END_LOADING })
    INV-->>U: Redirected to InvoiceDetails page
    end

    rect rgb(255, 245, 230)
    Note over U, SMTP: Phase 2: Send Invoice via Email
    U->>INV: Clicks "Send Invoice" on InvoiceDetails
    INV->>AX: POST /send-pdf { email, company, items, total, ... }
    AX->>EX: POST /send-pdf (no auth middleware)
    EX->>TPL: pdfTemplate(req.body) renders HTML string
    TPL-->>EX: Full HTML document with inline CSS
    EX->>PDF: pdf.create(html, { format: 'A4' }).toFile('invoice.pdf')
    PDF-->>EX: invoice.pdf written to disk
    EX->>SMTP: transporter.sendMail()
    Note over SMTP: From: Accountill hello@accountill.com<br/>To: client email<br/>ReplyTo: company email<br/>Attachment: invoice.pdf
    SMTP-->>EX: Delivery success
    EX-->>AX: 200 OK
    AX-->>INV: Success
    INV-->>U: "Invoice sent!" snackbar
    end
```

### Step-by-Step Walkthrough

| # | Step | What Happens | Data | Evidence |
|---|---|---|---|---|
| 1 | Form filled | User enters line items (`itemName`, `unitPrice`, `quantity`, `discount`), selects client, sets due date, currency | Initial state from `client/src/initialState.js:4–17` | `initialState.js:4–17` |
| 2 | Invoice number generated | `Math.floor(Math.random() * 100000)` — **not sequential**, potential for collisions | — | `initialState.js:13` |
| 3 | Redux action | `createInvoice(invoice, history)` dispatches `START_LOADING`, calls API, dispatches `ADD_NEW` | — | `actions/invoiceActions.js:42–52` |
| 4 | API call | `POST /invoices` with the full invoice object as JSON body | — | `api/index.js:16` |
| 5 | Controller | `const newInvoice = new InvoiceModel(invoice)` then `newInvoice.save()` | Writes: `invoicemodels` collection | `controllers/invoices.js:55–61` |
| 6 | **No validation** | Controller does **not** validate required fields, amounts, or check that the `creator` matches the authenticated user | — | `controllers/invoices.js:53–66` |
| 7 | Redirect | `history.push('/invoice/${data._id}')` navigates to InvoiceDetails | — | `actions/invoiceActions.js:47` |
| 8 | Send PDF | User clicks send button → `POST /send-pdf` with invoice data and recipient email | — | `server/index.js:53` |
| 9 | PDF render | `html-pdf` renders the `documents/index.js` template into an A4 PDF | — | `server/index.js:57` |
| 10 | File write | PDF saved to `${__dirname}/invoice.pdf` (single file, overwritten each time) | Writes: disk file | `server/index.js:57` |
| 11 | Email send | Nodemailer sends email with PDF attachment | — | `server/index.js:60–71` |
| 12 | **No auth on /send-pdf** | This endpoint has **no authentication** — anyone can call it | — | `server/index.js:53` |

### Failure Scenarios
| Failure | What Happens | Database State | Evidence |
|---|---|---|---|
| DB save fails | `409` status with error message returned | No invoice created | `controllers/invoices.js:63` |
| PDF generation fails | `res.send(Promise.reject())` — sends a rejected promise object (not a proper error response) | Invoice already saved in DB | `server/index.js:73–74` |
| SMTP failure | Email not delivered, but `res.send(Promise.resolve())` is called regardless (response sent before knowing if email succeeded) | Invoice exists, PDF on disk, email not delivered | `server/index.js:76` |
| Concurrent requests | Multiple users generating PDFs simultaneously would overwrite the same `invoice.pdf` file — **race condition** | Corrupt/mixed PDF attachments | `server/index.js:57` |

### Validation Checks Along the Path
| Check | Where | What it validates |
|---|---|---|
| JWT token presence | `client/src/api/index.js:7` (interceptor) | Attaches token if profile exists in localStorage |
| ObjectId validity (on update/delete only) | `controllers/invoices.js:85,96` | `mongoose.Types.ObjectId.isValid(_id)` |
| **No server-side input validation** | — | No check for required fields, valid numbers, valid email, etc. |
| **No ownership check** | — | No check that the invoice belongs to the requesting user |

---

## Key Observations from the Traces

1. **Signup creates a Profile automatically** from the client side — if this call fails, the user has no profile and the app may break.
2. **Invoice numbers are random**, not sequential — `Math.floor(Math.random() * 100000)` can produce duplicates.
3. **PDF generation uses a single shared file** on disk — concurrent requests will corrupt each other's PDFs.
4. **The `/send-pdf` endpoint has zero authentication** — any external caller could use it to send emails through the server's SMTP credentials.
5. **Error handling in the PDF/email pipeline is broken** — `res.send(Promise.reject())` and `res.send(Promise.resolve())` are not valid Express responses.
