# DocumentVault

A simple digital document locker built with the MERN stack. Users organize files into folders, upload documents, and track expiry for insurance and agreements. Expiry events can be exported as `.ics` files and imported into Google Calendar.

Junior-level MVC project. No OAuth or Google Calendar API.

| Item | Details |
|------|---------|
| Date of issue | 20 October 2026 |
| Version | 1.0 |
| Status | Draft for development |

---

## 1. Features

- Register, login, logout (JWT in HTTP-only cookie)
- Nested folders and documents (upload, view, download, update, delete)
- Two-level categories: pick a category (Personal, Company), then a subcategory using radio buttons
- Every document: `fileName`, `file`, `description`, `category`, `subcategory`, `issueDate` (defaults to today)
- Expiry subcategories (e.g. insurance, agreements): expiry date, reminder time, optional repeat every 3 / 6 / 12 hours. Reminder date is set automatically to one month before expiry
- Financial-year subcategories (e.g. income tax, accounts): financial year (`2025-26`)
- In-app notifications with unread count and mark-as-read
- `.ics` download for documents with an expiry date

---

## 2. SDLC Phases

1. **Requirements:** features and fields (Sections 1 and 3)
2. **Design:** MVC layout, data models, API (Sections 4 to 6)
3. **Implementation:** backend first, then frontend, one feature at a time
4. **Testing:** unit tests for utils, API tests with Supertest
5. **Deployment:** frontend build, backend on Node, MongoDB Atlas
6. **Maintenance:** bug fixes, new categories, `npm audit`

---

## 3. Categories and Document Fields

Documents use a two-level tree. The user picks a **category** first, then a **subcategory** from radio buttons, as in the Your e-Locker screen.

```
Personal
├── ( ) Passport Size Photos
├── ( ) PAN Card
├── ( ) Driving Licence
├── ( ) Insurance            [expiry fields]
└── ( ) Education
Company
├── ( ) Documents For QR
├── ( ) GST Registration
├── ( ) PF Certificate
├── ( ) DSC
├── ( ) Agreement            [expiry fields]
└── ( ) Income Tax Return    [financial year]
```

Each subcategory defines its extra fields. The server stores this in `config/categories.js`, so adding a subcategory requires no code change elsewhere.

| Field type | Applies to | Fields |
|------------|------------|--------|
| Base | All documents | `fileName`, `file`, `description`, `issueDate` (defaults to today) |
| Expiry | Subcategories marked expiry (e.g. Insurance, Agreement) | `expiryDate`, `reminderTime` (HH:mm), `repeatEveryHours` (3, 6, 12, optional). `reminderDate` is set automatically to one month before expiry |
| Financial year | Subcategories marked financial year (e.g. Income Tax, Accounts) | `financialYear` (format `YYYY-YY`) |

Server rules:
- The subcategory must belong to the chosen category.
- If the subcategory needs expiry, all expiry fields are required. Otherwise they are cleared.
- If the subcategory needs a financial year, it is required and validated. Otherwise it is cleared.

---

## 4. Architecture (MVC)

- **Model:** Mongoose schemas in `server/models/`
- **View:** React components in `client/src/`
- **Controller:** request logic in `server/controllers/`
- **Utils:** `ics.util.js`, `reminder.util.js`, `financialYear.util.js`
- **Job:** `reminder.job.js` (node-cron)

Request flow: React → Routes → Middleware → Controller → Model → MongoDB

---

## 5. Data Models

- **User:** name, email (unique), password (bcrypt hash), timezone
- **Folder:** name, owner, parent (null for root)
- **Document:** owner, folder, fileName, description, category, subcategory, issueDate (default: now), filePath, financialYear, expiry (expiryDate, reminderDate [auto], reminderTime, repeatEveryHours, nextReminderAt)
- **Notification:** user, document, title, message, isRead

---

## 6. Modules Used

**Backend**
- **express:** routes and middleware
- **mongoose:** MongoDB schemas and queries
- **dotenv:** loads `.env` secrets
- **bcryptjs:** hashes passwords
- **jsonwebtoken:** creates and verifies login tokens
- **cookie-parser:** reads the auth cookie
- **cors:** lets React call the API
- **helmet:** secure HTTP headers
- **express-rate-limit:** limits login attempts
- **multer:** handles file uploads
- **node-cron:** runs the reminder job
- **jest, supertest:** unit and API tests

**Frontend:** react, react-dom (UI), react-router-dom (pages), axios (API calls), vite (dev server and build)

---

## 7. Key Logic

**Reminder date:** calculated as expiry date minus one month. Example: expiry `2026-12-15` gives reminder `2026-11-15`. If the previous month is shorter, use its last day (expiry `2027-03-31` gives `2027-02-28`). If that date is already past when the document is saved, use today.

**Repeat reminders:** start at `reminderDate` + `reminderTime`. Add `repeatEveryHours` for each next reminder. Stop once the next time passes `expiryDate`.

**Reminder job (every minute):**
1. Find documents where `expiry.nextReminderAt <= now`.
2. Update `nextReminderAt` only if it still matches the value read. This prevents duplicate notifications.
3. Create a notification for the owner.

**ICS export:** `GET /api/documents/:id/ics` returns a `text/calendar` file. Use a stable `UID` so re-importing updates the event instead of duplicating it. Import in Google Calendar under Settings → Import & export.

**Security:** bcrypt for passwords, JWT in an HTTP-only cookie, and every query filters by `req.user.id`.

---

## 8. API Summary

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/register`, `/api/auth/login` | Sign up / log in |
| GET, POST | `/api/folders` | List tree / create folder |
| GET, POST | `/api/documents` | List / upload (multipart) |
| GET, PATCH, DELETE | `/api/documents/:id` | View / update / delete |
| GET | `/api/documents/:id/download` | Download file |
| GET | `/api/documents/:id/ics` | Download calendar event |
| GET | `/api/notifications` | List notifications |
| GET | `/api/notifications/unread-count` | Unread count |
| PATCH | `/api/notifications/:id/read` | Mark as read |

---

## 9. Testing

Key cases: a December expiry gets a November reminder, duplicate email returns 409, Insurance subcategory without `expiryDate` returns 400, another user's document returns 404, and a reminder processed twice creates one notification. Run with `cd server && npm test`.

---

## 10. Setup

**Backend:** `cd server && npm install`. Create `server/.env`:
```
PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/documentvault
JWT_SECRET=replace_with_a_long_random_string
CLIENT_URL=http://localhost:5173
```
Run with `npm run dev`.

**Frontend:** `cd client && npm install`. Create `client/.env`:
```
VITE_API_URL=http://localhost:5000/api
```
Run with `npm run dev`, then open `http://localhost:5173`.

---

## 11. Limitations and Future Work

- Calendar export is one-way. Changed dates need a new `.ics` import.
- Reminders can be up to one minute late.
- Files are stored on local disk. Use cloud storage in production.
- Future: Google Calendar API sync, email reminders, S3 storage, document search.
