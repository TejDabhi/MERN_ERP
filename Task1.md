# DocumentVault

A simple digital document locker built with the MERN stack (MongoDB, Express.js, React, Node.js). Users sign up, organize files into folders, upload documents with a description, and track expiry-based documents such as insurance and agreements. Reminders can be sent in-app, and expiry events can be exported as `.ics` files to import into Google Calendar.

This project is designed as a junior-level project. It uses a simple MVC structure and avoids advanced patterns such as microservices, message queues, or OAuth integrations.

---

## 1. Project Overview

| Item | Details |
|------|---------|
| Project name | DocumentVault |
| Type | Web application (single repository, client and server) |
| Stack | MongoDB, Express.js, React, Node.js |
| Architecture | MVC on the backend, component-based UI on the frontend |
| Calendar integration | `.ics` file export (one-way, no Google API) |
| Target users | Individuals who store personal and financial documents |

### 1.1 Goals
- Let users store documents securely and find them easily.
- Support category-specific fields (expiry for insurance and agreements, financial year for tax and accounts).
- Send reminders at a chosen date and time, or repeatedly at a fixed interval.
- Let users add expiry events to Google Calendar through `.ics` files.

### 1.2 Non-Goals (for this version)
- Google Calendar API sync (OAuth, two-way sync).
- Document sharing between users.
- Full-text search inside file contents.
- OCR, digital signatures, or version history.
- Microservices, Redis, or message queues.

---

## 2. Software Development Life Cycle (SDLC)

This project follows a simple **Waterfall-style SDLC with iterative milestones**. Each phase has defined inputs and outputs.

### Phase 1: Requirement Analysis
**Purpose:** Decide what the system must do.

Functional requirements:
- FR-01: User registration, login, and logout.
- FR-02: Create, rename, and delete folders (folders can contain subfolders).
- FR-03: Upload a document with a file name, the file itself, and a description.
- FR-04: View, download, update, and delete a document.
- FR-05: Category selection for each document (see Section 4).
- FR-06: Insurance and agreement documents store expiry date, reminder date, reminder time, and an optional repeat interval (3, 6, or 12 hours).
- FR-07: Income tax and accounts documents store a financial year (for example `2025-26`).
- FR-08: In-app notifications with unread count and mark-as-read.
- FR-09: Scheduled reminders created by a cron job.
- FR-10: Download an `.ics` file for any document that has an expiry date.

Non-functional requirements:
- NFR-01: Passwords are hashed. Routes are protected with JWT.
- NFR-02: Users can only access their own documents.
- NFR-03: Upload size and file types are restricted.
- NFR-04: Each API has basic test coverage.
- NFR-05: Code is organized so features are easy to find and change.

### Phase 2: System Design
**Purpose:** Plan the structure before writing code.

- Architecture: MVC on the backend (Model, View, Controller), React on the frontend.
- Database: MongoDB with Mongoose schemas.
- Authentication: JWT stored in an HTTP-only cookie.
- File storage: files saved on the server disk in an `uploads/` folder, with the database storing only the metadata and file path.
- Scheduling: `node-cron` runs every minute to check for due reminders.
- Calendar: the server generates `.ics` text, and the browser downloads it.

### Phase 3: Implementation
**Purpose:** Build the features in small steps.

Recommended order:
1. Backend setup: Express, MongoDB connection, error handler.
2. Auth: register, login, JWT middleware.
3. Folders: create, list tree, rename, delete.
4. Documents: upload, list, view metadata, download, update, delete.
5. Category rules: expiry fields and financial year validation.
6. Reminders and notifications: cron job, notification model, unread count.
7. `.ics` export.
8. Frontend: auth pages, folder tree, document list, forms, notification bell.
9. Testing and cleanup.

### Phase 4: Testing
**Purpose:** Confirm that the software works as specified.

- **Unit tests:** validators, date helpers, `.ics` generator, reminder-time calculation.
- **Integration tests (API):** auth flow, folder and document CRUD, access control, upload rules.
- **Manual testing:** full user flow in the browser.
- Test cases are described in Section 9.

### Phase 5: Deployment
**Purpose:** Run the application for real users.

- Frontend: build with `npm run build` and serve from a static host or from Express.
- Backend: run with `node` or a process manager such as PM2.
- Database: MongoDB Atlas (free tier) or a local MongoDB instance.
- Environment variables are set in a `.env` file and never committed to Git.

### Phase 6: Maintenance
**Purpose:** Keep the system working after release.

- Fix bugs reported by users.
- Monitor the reminder cron job logs.
- Add new document categories by updating the category config (Section 4).
- Review dependencies regularly with `npm audit`.

---

## 3. Document Types and Fields

Every document has the same base fields. Some categories add extra fields.

### 3.1 Base fields (all documents)

| Field | Required | Notes |
|-------|----------|-------|
| `fileName` | Yes | Display name, max 150 characters |
| `file` | Yes | The uploaded file (PDF, JPG, PNG, DOCX, max 10 MB) |
| `description` | No | Free text, max 500 characters |
| `category` | Yes | One of the categories in Section 3.2 |
| `folderId` | Yes | The folder the document belongs to |

### 3.2 Categories

| Category | Extra fields | Expiry and reminders |
|----------|--------------|----------------------|
| `identity` | None | No |
| `education` | None | No |
| `property` | None | No |
| `other` | None | No |
| `insurance` | Expiry section | Yes |
| `agreement` | Expiry section | Yes |
| `income_tax` | Financial year | No |
| `accounts` | Financial year | No |

### 3.3 Expiry section (insurance and agreement only)

| Field | Required | Example | Notes |
|-------|----------|---------|-------|
| `expiryDate` | Yes | `2027-03-31` | Date the document expires |
| `reminderDate` | Yes | `2027-03-01` | First date a reminder is sent |
| `reminderTime` | Yes | `09:00` | 24-hour time, user's local time |
| `repeatEveryHours` | No | `3`, `6`, or `12` | If set, reminders repeat at this interval until expiry |

**How repeat works:** Starting from `reminderDate` and `reminderTime`, a reminder is sent. If `repeatEveryHours` is set, the next reminder is scheduled that many hours later. Repeats stop once the next reminder would fall after `expiryDate`.

Example: `reminderDate = 2027-03-01`, `reminderTime = 09:00`, `repeatEveryHours = 6`, so reminders fire at 09:00, 15:00, 21:00, and so on, until `expiryDate`.

### 3.4 Financial year section (income tax and accounts only)

| Field | Required | Example | Validation |
|-------|----------|---------|------------|
| `financialYear` | Yes | `2025-26` | Format `YYYY-YY`, where the second part equals the last two digits of the next year |

---

## 4. Architecture (MVC)

```
Browser (React)
     |
     |  HTTP (JSON, multipart for uploads)
     v
Express Server
  Routes  ->  Middleware  ->  Controllers  ->  Models  ->  MongoDB
                                  |
                                  +-> Utils (ics, dates, file helpers)
                                  +-> Cron job (reminders) -> Notification model
```

### 4.1 MVC roles in this project

| Layer | Folder | Responsibility |
|-------|--------|----------------|
| Model | `server/src/models/` | Defines the MongoDB schema and data rules |
| View | `client/src/` (React) | Displays data and collects user input |
| Controller | `server/src/controllers/` | Handles requests, calls models, returns responses |
| Routes | `server/src/routes/` | Maps URLs to controller functions |
| Middleware | `server/src/middleware/` | Auth check, upload handling, error handling |
| Utils | `server/src/utils/` | Helper functions such as date handling and `.ics` generation |
| Jobs | `server/src/jobs/` | Scheduled tasks (reminder cron) |

Note: In a React-based MERN app, React acts as the View. The server acts as Model and Controller, which is a common junior-level interpretation of MVC.

---

## 5. Folder Structure

```
documentvault/
├── README.md
├── .gitignore
├── client/                          # React frontend (View)
│   ├── package.json
│   ├── .env
│   ├── public/
│   └── src/
│       ├── main.jsx
│       ├── App.jsx
│       ├── api/
│       │   └── axiosClient.js       # Axios instance with base URL and credentials
│       ├── context/
│       │   └── AuthContext.jsx      # Stores logged-in user
│       ├── components/
│       │   ├── FolderTree.jsx
│       │   ├── DocumentList.jsx
│       │   ├── DocumentForm.jsx     # Shows extra fields based on category
│       │   ├── NotificationBell.jsx
│       │   └── ProtectedRoute.jsx
│       ├── pages/
│       │   ├── LoginPage.jsx
│       │   ├── RegisterPage.jsx
│       │   ├── DashboardPage.jsx
│       │   └── NotificationsPage.jsx
│       └── styles/
│           └── index.css
└── server/                          # Express backend (Model + Controller)
    ├── package.json
    ├── .env
    ├── server.js                    # Entry point: starts server and cron
    ├── app.js                       # Express app setup
    ├── config/
    │   ├── db.js                    # MongoDB connection
    │   └── categories.js            # Category rules (fields required per category)
    ├── models/
    │   ├── User.model.js
    │   ├── Folder.model.js
    │   ├── Document.model.js
    │   └── Notification.model.js
    ├── controllers/
    │   ├── auth.controller.js
    │   ├── folder.controller.js
    │   ├── document.controller.js
    │   ├── notification.controller.js
    │   └── calendar.controller.js
    ├── routes/
    │   ├── index.js
    │   ├── auth.routes.js
    │   ├── folder.routes.js
    │   ├── document.routes.js
    │   ├── notification.routes.js
    │   └── calendar.routes.js
    ├── middleware/
    │   ├── auth.middleware.js
    │   ├── upload.middleware.js
    │   ├── validate.middleware.js
    │   └── error.middleware.js
    ├── utils/
    │   ├── ics.util.js              # Builds .ics text
    │   ├── reminder.util.js         # Calculates next reminder time
    │   └── financialYear.util.js    # Validates "YYYY-YY"
    ├── jobs/
    │   └── reminder.job.js          # Cron job
    ├── uploads/                     # Stored files (not committed to Git)
    └── tests/
        ├── auth.test.js
        ├── folder.test.js
        ├── document.test.js
        ├── calendar.test.js
        └── reminder.util.test.js
```

---

## 6. Data Models

### 6.1 User

| Field | Type | Notes |
|-------|------|-------|
| `name` | String | Required |
| `email` | String | Required, unique, lowercase |
| `password` | String | Hashed with bcrypt, never returned in responses |
| `timezone` | String | For example `Asia/Kolkata`, used for reminders and `.ics` |
| `createdAt` | Date | Automatic |

### 6.2 Folder

| Field | Type | Notes |
|-------|------|-------|
| `name` | String | Required |
| `owner` | ObjectId (User) | Required, indexed |
| `parent` | ObjectId (Folder) | `null` for root folders |
| `createdAt` | Date | Automatic |

Subfolders are stored as folders that point to a parent. The tree is built in the controller.

### 6.3 Document

| Field | Type | Notes |
|-------|------|-------|
| `owner` | ObjectId (User) | Required, indexed |
| `folder` | ObjectId (Folder) | Required |
| `fileName` | String | Required |
| `description` | String | Optional |
| `category` | String | Enum from Section 3.2 |
| `filePath` | String | Path on server disk, never sent to the client |
| `originalName` | String | Name of the uploaded file |
| `mimeType` | String | Checked on upload |
| `size` | Number | Bytes |
| `expiry` | Object | Used only for insurance and agreement |
| `expiry.expiryDate` | Date | Optional |
| `expiry.reminderDate` | Date | Optional |
| `expiry.reminderTime` | String | `HH:mm` |
| `expiry.repeatEveryHours` | Number | `3`, `6`, `12`, or `null` |
| `expiry.nextReminderAt` | Date | Next time the cron job should send a reminder |
| `financialYear` | String | Used only for income tax and accounts |
| `createdAt` / `updatedAt` | Date | Automatic |

### 6.4 Notification

| Field | Type | Notes |
|-------|------|-------|
| `user` | ObjectId (User) | Required, indexed |
| `document` | ObjectId (Document) | Optional |
| `title` | String | Short heading |
| `message` | String | Body text |
| `isRead` | Boolean | Default `false` |
| `createdAt` | Date | Automatic |

Unread count is calculated with `Notification.countDocuments({ user, isRead: false })`.

---

## 7. Modules Explained

This section explains each library used, what it does, and where it appears in the project.

### 7.1 Backend Modules

**express**
- Web framework for Node.js. It handles routing, request parsing, and middleware.
- Used in `app.js` and in every file under `routes/`.
- Example: `app.use('/api/documents', documentRoutes);`

**mongoose**
- Object Data Modeling (ODM) library for MongoDB. It defines schemas, validates data, and provides query methods.
- Used in `config/db.js` to connect, and in every file under `models/`.
- Example: `const Document = mongoose.model('Document', documentSchema);`

**dotenv**
- Loads variables from a `.env` file into `process.env`.
- Used at the top of `server.js`: `require('dotenv').config();`
- Keeps secrets such as `JWT_SECRET` and `MONGO_URI` out of source code.

**bcryptjs**
- Hashes passwords with a salt so plain-text passwords are never stored.
- Used in `auth.controller.js` during registration and login.
- Example: `const hash = await bcrypt.hash(password, 10);`

**jsonwebtoken**
- Creates and verifies JSON Web Tokens for login sessions.
- Used in `auth.controller.js` to sign the token and in `auth.middleware.js` to verify it.
- Example: `jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '1d' });`

**cookie-parser**
- Reads cookies from incoming requests.
- Used in `app.js` so the auth middleware can read the JWT from the cookie.

**cors**
- Allows the React app (different port) to call the Express API.
- Used in `app.js` with `credentials: true` and the client origin.

**helmet**
- Adds security-related HTTP headers.
- Used in `app.js` as `app.use(helmet());`

**express-rate-limit**
- Limits how many requests one IP can make in a time window.
- Used on login and register routes to slow down brute-force attempts.

**multer**
- Handles `multipart/form-data` uploads.
- Used in `upload.middleware.js`. It stores files in `server/uploads/` and checks file type and size.
- Example: `multer({ storage, limits: { fileSize: 10 * 1024 * 1024 }, fileFilter })`

**express-validator** (or Joi)
- Validates request bodies, such as required fields, date formats, and allowed categories.
- Used in `validate.middleware.js` and attached to routes.

**node-cron**
- Runs a function on a schedule using cron syntax.
- Used in `jobs/reminder.job.js`. The schedule `* * * * *` runs every minute.
- Example: `cron.schedule('* * * * *', checkDueReminders);`

**nodemailer** (optional)
- Sends emails. Use it only if you want email reminders in addition to in-app notifications.
- Used inside the reminder job.

**ics** (npm package, optional)
- Creates `.ics` calendar files from event objects.
- If you prefer no dependency, `utils/ics.util.js` can build the text manually, as shown in Section 8.3.

**jest** and **supertest** (testing)
- `jest` runs test cases and assertions.
- `supertest` sends HTTP requests to the Express app during tests without starting a real server.
- Used in `tests/`.

**mongodb-memory-server** (testing, optional)
- Starts a temporary in-memory MongoDB for tests, so tests do not touch your real database.

### 7.2 Frontend Modules

**react**
- Library for building user interfaces from components.
- Used for every file under `components/` and `pages/`.

**react-dom**
- Connects React to the browser DOM. Used in `main.jsx`.

**react-router-dom**
- Handles client-side navigation between pages.
- Used in `App.jsx` to define routes such as `/login`, `/dashboard`, and `/notifications`.

**axios**
- Makes HTTP requests to the backend.
- Configured once in `api/axiosClient.js` with a base URL and `withCredentials: true` so cookies are sent.

**vite**
- Development server and build tool for React. It starts quickly and rebuilds on save.

**Optional UI helpers:** Bootstrap or Tailwind CSS for styling, and `react-hot-toast` for success and error messages.

---

## 8. Key Logic Explained

### 8.1 Category validation (`config/categories.js`)

A single config file defines which fields each category needs. The controller reads it, so you never need to hard-code category checks in many places.

```js
// server/src/config/categories.js
module.exports = {
  identity:   { requiresExpiry: false, requiresFinancialYear: false },
  education:  { requiresExpiry: false, requiresFinancialYear: false },
  property:   { requiresExpiry: false, requiresFinancialYear: false },
  other:      { requiresExpiry: false, requiresFinancialYear: false },
  insurance:  { requiresExpiry: true,  requiresFinancialYear: false },
  agreement:  { requiresExpiry: true,  requiresFinancialYear: false },
  income_tax: { requiresExpiry: false, requiresFinancialYear: true },
  accounts:   { requiresExpiry: false, requiresFinancialYear: true },
};
```

Server rule: if a category requires expiry, all expiry fields must be present. If it does not, expiry fields are cleared before saving. The same applies to `financialYear`.

### 8.2 Reminder calculation (`utils/reminder.util.js`)

Converts `reminderDate` and `reminderTime` into a single `Date` in the user's timezone, then calculates the next reminder.

```js
// server/src/utils/reminder.util.js
const ALLOWED_REPEATS = [3, 6, 12];

function buildReminderDate(reminderDate, reminderTime) {
  // reminderDate: "2027-03-01", reminderTime: "09:00"
  // Note: production code should use a timezone-aware library such as luxon
  // so the time is interpreted in the user's timezone.
  const [hours, minutes] = reminderTime.split(':').map(Number);
  const date = new Date(reminderDate);
  date.setHours(hours, minutes, 0, 0);
  return date;
}

function getNextReminder(currentReminderAt, repeatEveryHours, expiryDate) {
  if (!repeatEveryHours) return null; // one-time reminder
  if (!ALLOWED_REPEATS.includes(repeatEveryHours)) {
    throw new Error('repeatEveryHours must be 3, 6, or 12');
  }
  const next = new Date(currentReminderAt.getTime() + repeatEveryHours * 60 * 60 * 1000);
  return next <= expiryDate ? next : null; // stop after expiry
}

module.exports = { buildReminderDate, getNextReminder };
```

For junior projects, `luxon` (`DateTime.fromISO(..., { zone: user.timezone })`) is the safer choice for timezone handling. The code above is simplified for clarity.

### 8.3 Reminder job (`jobs/reminder.job.js`)

Runs every minute. It finds documents whose `nextReminderAt` is due, creates an in-app notification, moves `nextReminderAt` forward or clears it, and does this in a way that prevents duplicates.

```js
// server/src/jobs/reminder.job.js
const cron = require('node-cron');
const Document = require('../models/Document.model');
const Notification = require('../models/Notification.model');
const { getNextReminder } = require('../utils/reminder.util');

async function processDueReminders() {
  const now = new Date();
  const due = await Document.find({
    'expiry.nextReminderAt': { $lte: now },
  }).limit(100);

  for (const doc of due) {
    // Atomic update: only proceed if nextReminderAt is still the value we read.
    // This prevents duplicate notifications if the job overlaps with itself.
    const scheduledAt = doc.expiry.nextReminderAt;
    const next = getNextReminder(scheduledAt, doc.expiry.repeatEveryHours, doc.expiry.expiryDate);

    const updated = await Document.findOneAndUpdate(
      { _id: doc._id, 'expiry.nextReminderAt': scheduledAt },
      { $set: { 'expiry.nextReminderAt': next } }
    );
    if (!updated) continue; // another run already handled it

    await Notification.create({
      user: doc.owner,
      document: doc._id,
      title: `${doc.fileName} is due soon`,
      message: `Expiry date: ${doc.expiry.expiryDate.toDateString()}`,
    });
  }
}

function startReminderJob() {
  cron.schedule('* * * * *', () => {
    processDueReminders().catch(err => console.error('Reminder job failed:', err));
  });
}

module.exports = { startReminderJob, processDueReminders };
```

Duplicate prevention in this design:
1. The reminder time is stored in `nextReminderAt`, so each reminder has one scheduled time.
2. The update is conditional on the old `nextReminderAt` value. Only one run can win.
3. Once a reminder is processed, `nextReminderAt` moves to the next time or becomes `null`.

### 8.4 `.ics` export (`utils/ics.util.js`)

An `.ics` file is plain text in the iCalendar format. The browser downloads it, and the user imports it into Google Calendar.

```js
// server/src/utils/ics.util.js
function formatUtc(date) {
  // Format: 20270301T090000Z
  return date.toISOString().replace(/[-:]/g, '').split('.')[0] + 'Z';
}

function buildIcs({ uid, title, description, start, end }) {
  const lines = [
    'BEGIN:VCALENDAR',
    'VERSION:2.0',
    'PRODID:-//DocumentVault//EN',
    'CALSCALE:GREGORIAN',
    'BEGIN:VEVENT',
    `UID:${uid}@documentvault`,
    `DTSTAMP:${formatUtc(new Date())}`,
    `DTSTART:${formatUtc(start)}`,
    `DTEND:${formatUtc(end)}`,
    `SUMMARY:${title}`,
    `DESCRIPTION:${description}`,
    'BEGIN:VALARM',
    'ACTION:DISPLAY',
    'DESCRIPTION:Reminder',
    'TRIGGER:-PT0M',
    'END:VALARM',
    'END:VEVENT',
    'END:VCALENDAR',
  ];
  return lines.join('\r\n');
}

module.exports = { buildIcs };
```

Notes:
- `UID` must be stable for the same document so re-importing updates the event instead of duplicating it.
- Commas, semicolons, and line breaks in `SUMMARY` and `DESCRIPTION` must be escaped (`\,` `\;` `\n`). Add an escape function in production code.
- Use `\r\n` line endings, as required by the iCalendar format.

**Endpoint:** `GET /api/documents/:id/ics` returns the file with the header `Content-Type: text/calendar` and `Content-Disposition: attachment; filename="document.ics"`.

**Import into Google Calendar:** Google Calendar on the web → Settings → Import & export → Select file → Import. Import is one-way. If the user changes the expiry date later, they must download and import a new `.ics` file.

### 8.5 Authentication flow

1. User registers. Password is hashed with bcrypt and the user is saved.
2. User logs in. Password is compared with bcrypt. A JWT is created and stored in an HTTP-only cookie.
3. Protected routes pass through `auth.middleware.js`, which verifies the JWT and sets `req.user`.
4. Controllers always filter by `req.user.id`, so users can only see their own data.

### 8.6 File upload flow

1. Client sends `multipart/form-data` with `file`, `fileName`, `description`, `category`, `folderId`, and any category-specific fields.
2. `upload.middleware.js` (multer) checks size and MIME type and saves the file to `uploads/` with a random name.
3. `document.controller.js` validates the category rules and saves metadata to MongoDB.
4. If the document is insurance or agreement with an expiry date, `nextReminderAt` is calculated from `reminderDate`, `reminderTime`, and the repeat setting.
5. On delete, the controller removes the database record and deletes the file from disk.

Stored file names use random values (for example, `crypto.randomUUID()`), so users cannot guess other files by URL.

---

## 9. API Reference

Base URL: `/api`. Protected routes require the JWT cookie.

### 9.1 Auth

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/auth/register` | No | Create account |
| POST | `/auth/login` | No | Log in, set cookie |
| POST | `/auth/logout` | Yes | Clear cookie |
| GET | `/auth/me` | Yes | Get current user |

### 9.2 Folders

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/folders` | Yes | Get folder tree |
| POST | `/folders` | Yes | Create folder (`name`, optional `parent`) |
| PATCH | `/folders/:id` | Yes | Rename folder |
| DELETE | `/folders/:id` | Yes | Delete folder (only if empty) |

### 9.3 Documents

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/documents?folder=:id&category=:cat` | Yes | List documents with filters |
| POST | `/documents` | Yes | Upload document (multipart) |
| GET | `/documents/:id` | Yes | Get document details |
| GET | `/documents/:id/download` | Yes | Download the file |
| PATCH | `/documents/:id` | Yes | Update fields (not the file) |
| DELETE | `/documents/:id` | Yes | Delete document and file |
| GET | `/documents/:id/ics` | Yes | Download `.ics` for expiry event |

### 9.4 Notifications

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/notifications` | Yes | List notifications |
| GET | `/notifications/unread-count` | Yes | Get unread count |
| PATCH | `/notifications/:id/read` | Yes | Mark one as read |
| PATCH | `/notifications/read-all` | Yes | Mark all as read |

### 9.5 Example request: upload an insurance document

```
POST /api/documents
Content-Type: multipart/form-data

file:             policy.pdf
fileName:         Car Insurance 2027
description:      Comprehensive policy, ICICI Lombard
category:         insurance
folderId:         64f1a2b3c4d5e6f7a8b9c0d1
expiryDate:       2027-03-31
reminderDate:     2027-03-01
reminderTime:     09:00
repeatEveryHours: 6
```

### 9.6 Example request: upload an income tax document

```
POST /api/documents
Content-Type: multipart/form-data

file:          itr-acknowledgement.pdf
fileName:      ITR Acknowledgement
description:   Filed online
category:      income_tax
folderId:      64f1a2b3c4d5e6f7a8b9c0d2
financialYear: 2025-26
```

### 9.7 Standard error response

```json
{
  "success": false,
  "message": "expiryDate is required for category insurance"
}
```

---

## 10. Testing Plan

### 10.1 Test types

| Type | Tool | What it covers |
|------|------|----------------|
| Unit | Jest | `reminder.util.js`, `ics.util.js`, `financialYear.util.js` |
| Integration (API) | Jest + Supertest | Routes, controllers, middleware, database |
| Manual | Browser | Full user flow |

### 10.2 Sample test cases

| ID | Area | Input | Expected result |
|----|------|-------|-----------------|
| TC-01 | Register | Valid name, email, password | 201, user created, password hashed |
| TC-02 | Register | Duplicate email | 409 error |
| TC-03 | Login | Wrong password | 401 error |
| TC-04 | Protected route | No cookie | 401 error |
| TC-05 | Folder | Create subfolder with valid parent | 201, parent set |
| TC-06 | Upload | Insurance without expiryDate | 400 error |
| TC-07 | Upload | Income tax with financialYear `2025-27` | 400 error (invalid format) |
| TC-08 | Upload | Income tax with financialYear `2025-26` | 201 |
| TC-09 | Upload | Identity document with expiryDate | Expiry fields ignored and cleared |
| TC-10 | Upload | File larger than 10 MB | 413 or 400 error |
| TC-11 | Upload | `.exe` file | 400 error (type not allowed) |
| TC-12 | Access | User B requests user A's document | 404 error |
| TC-13 | Reminder util | `repeatEveryHours = 6`, start 09:00 | Next reminder 15:00 |
| TC-14 | Reminder util | `repeatEveryHours = 12`, next past expiry | Returns `null` |
| TC-15 | Reminder job | Due reminder processed twice | Only one notification created |
| TC-16 | ICS | Valid document | Response contains `BEGIN:VCALENDAR` and `UID` |
| TC-17 | Notifications | Mark as read | `isRead` becomes `true`, unread count decreases |

### 10.3 Running tests

```bash
cd server
npm test
```

---

## 11. Setup Instructions

### 11.1 Prerequisites
- Node.js 18 or later
- MongoDB (local install or MongoDB Atlas)
- Git
- A code editor such as VS Code

### 11.2 Backend setup

```bash
cd server
npm install
```

Create `server/.env`:

```
PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/documentvault
JWT_SECRET=replace_with_a_long_random_string
CLIENT_URL=http://localhost:5173
```

Start the server:

```bash
npm run dev
```

### 11.3 Frontend setup

```bash
cd client
npm install
```

Create `client/.env`:

```
VITE_API_URL=http://localhost:5000/api
```

Start the client:

```bash
npm run dev
```

Open `http://localhost:5173` in your browser.

### 11.4 Suggested npm scripts

`server/package.json`:
```json
{
  "scripts": {
    "dev": "nodemon server.js",
    "start": "node server.js",
    "test": "jest --runInBand"
  }
}
```

---

## 12. Security Basics

- Passwords are hashed with bcrypt. Plain-text passwords are never stored or logged.
- JWT is stored in an HTTP-only, `SameSite` cookie, so JavaScript cannot read it.
- `helmet` sets secure HTTP headers. `express-rate-limit` slows down login attempts.
- Every document query filters by `owner`, so users cannot access each other's files.
- Uploaded files are stored with random names outside the public web root, and downloads go through an authenticated route.
- File type and size are checked on the server, not only in the browser.
- `.env` and `uploads/` are listed in `.gitignore`.
- Input is validated before it reaches the database.

---

## 13. Limitations

- Calendar sync is one-way (export only). Changes made later need a new `.ics` import.
- Reminders are checked once per minute, so a reminder may be up to a minute late.
- Files are stored on the local disk. For production, move files to cloud storage such as AWS S3.
- Timezone handling is simplified. Use `luxon` for accurate timezone support.
- No email sending in the base version. Add `nodemailer` later if needed.

---

## 14. Future Enhancements

- Two-way Google Calendar sync using the Google Calendar API.
- Email reminders through `nodemailer`.
- Cloud file storage (S3).
- Document search by name and description.
- Document sharing with other users.
- Mobile-friendly layout or a React Native app.

---

## 15. Glossary

| Term | Meaning |
|------|---------|
| MERN | MongoDB, Express, React, Node.js |
| MVC | Model-View-Controller, a way to separate data, UI, and logic |
| JWT | JSON Web Token, a signed token used for login sessions |
| ODM | Object Data Modeling, a layer that maps code objects to database documents |
| Cron | A scheduler that runs tasks at fixed times |
| `.ics` | iCalendar file format, used to share calendar events |
| Middleware | Code that runs between a request and the controller |
| Multipart | A request format that carries files along with form fields |

---

## 16. License

This project is for learning purposes. Add a license file (for example MIT) if you plan to share it publicly.
