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
- Every document: `fileName`, `file`, `description`, `category`, `issueDate` (defaults to today)
- Insurance and agreement: expiry date, reminder time, optional repeat every 3 / 6 / 12 hours (reminder date is set automatically to one month before expiry)
- Income tax and accounts: financial year (`2025-26`)
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

## 3. Document Fields

| Category | Extra fields |
|----------|--------------|
| All categories | `issueDate` (defaults to current date, can be changed) |
| identity, education, property, other | None |
| insurance, agreement | `expiryDate`, `reminderTime` (HH:mm), `repeatEveryHours` (3, 6, 12). `reminderDate` is auto-set to one month before expiry |
| income_tax, accounts | `financialYear` (format `YYYY-YY`) |

If a category needs expiry, the server requires all expiry fields. Otherwise it clears them. `issueDate` is set to the upload date when the user does not enter one.

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
- **Document:** owner, folder, fileName, description, category, issueDate (default: now), filePath, financialYear, expiry (expiryDate, reminderDate [auto], reminderTime, repeatEveryHours, nextReminderAt)
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

**Frontend**
- **react, react-dom:** builds the UI
- **react-router-dom:** page navigation
- **axios:** HTTP calls to the API
- **vite:** dev server and build

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

Key cases: a December expiry gets a November reminder, duplicate email returns 409, insurance without `expiryDate` returns 400, another user's document returns 404, and a reminder processed twice creates one notification. Run with `cd server && npm test`.

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

---

## 12. License

For learning purposes.
