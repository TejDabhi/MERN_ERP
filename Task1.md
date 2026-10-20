# DocumentVault

A simple digital document locker built with the MERN stack. Users register, organize files into folders, upload documents, and track expiry for insurance and agreements. Expiry events can be exported as `.ics` files and imported into Google Calendar.

Junior-level project using an MVC structure. No OAuth, no Google Calendar API, no microservices.

---

## 1. Features

- Register, login, logout (JWT in HTTP-only cookie)
- Nested folders (folder → subfolder)
- Upload, view, download, update, delete documents
- Every document: `fileName`, `file`, `description`, `category`
- Insurance and agreement: expiry date, reminder date, reminder time, optional repeat every 3 / 6 / 12 hours
- Income tax and accounts: financial year (`2025-26`)
- In-app notifications with unread count and mark-as-read
- Reminder cron job (runs every minute)
- Download `.ics` for any document with an expiry date

---

## 2. SDLC Phases

| Phase | What happens in this project |
|-------|------------------------------|
| Requirements | List features and fields (Section 1 and 3) |
| Design | MVC layout, data models, API routes (Sections 4 and 6) |
| Implementation | Backend first, then frontend, feature by feature |
| Testing | Unit tests for utils, API tests with Supertest |
| Deployment | Frontend build, backend on Node, database on MongoDB Atlas |
| Maintenance | Fix bugs, update categories, run `npm audit` |

---

## 3. Document Fields

| Category | Extra fields |
|----------|--------------|
| identity, education, property, other | None |
| insurance, agreement | `expiryDate`, `reminderDate`, `reminderTime` (HH:mm), `repeatEveryHours` (optional: 3, 6, 12) |
| income_tax, accounts | `financialYear` (format `YYYY-YY`) |

Server rule: if a category needs expiry, all expiry fields are required. Otherwise they are cleared before saving.

---

## 4. Architecture (MVC)

```
React (View) → Express Routes → Middleware → Controllers → Models → MongoDB
                                                  ↓
                                   Utils (ics, reminder, financial year)
                                   Cron job → Notification model
```

- **Model:** Mongoose schemas in `server/src/models/`
- **View:** React components in `client/src/`
- **Controller:** request logic in `server/src/controllers/`

---

## 5. Folder Structure

```
documentvault/
├── client/
│   └── src/
│       ├── api/axiosClient.js
│       ├── components/   (FolderTree, DocumentList, DocumentForm, NotificationBell)
│       ├── pages/        (Login, Register, Dashboard, Notifications)
│       └── App.jsx
└── server/
    ├── server.js
    ├── app.js
    ├── config/           (db.js, categories.js)
    ├── models/           (User, Folder, Document, Notification)
    ├── controllers/      (auth, folder, document, notification, calendar)
    ├── routes/           (one file per resource)
    ├── middleware/       (auth, upload, validate, error)
    ├── utils/            (ics.util.js, reminder.util.js, financialYear.util.js)
    ├── jobs/             (reminder.job.js)
    ├── uploads/          (not committed to Git)
    └── tests/
```

---

## 6. Data Models

- **User:** name, email (unique), password (bcrypt hash), timezone
- **Folder:** name, owner, parent (null for root)
- **Document:** owner, folder, fileName, description, category, filePath, mimeType, size, financialYear, expiry (expiryDate, reminderDate, reminderTime, repeatEveryHours, nextReminderAt)
- **Notification:** user, document, title, message, isRead

---

## 7. Modules Used

**Backend**

| Module | Purpose |
|--------|---------|
| express | Routing and middleware |
| mongoose | MongoDB schemas and queries |
| dotenv | Loads `.env` secrets |
| bcryptjs | Hashes passwords |
| jsonwebtoken | Creates and verifies login tokens |
| cookie-parser | Reads auth cookie |
| cors | Allows React app to call the API |
| helmet | Secure HTTP headers |
| express-rate-limit | Limits login attempts |
| multer | Handles file uploads |
| express-validator | Validates request data |
| node-cron | Runs reminder job on a schedule |
| jest, supertest | Unit and API tests |

**Frontend**

| Module | Purpose |
|--------|---------|
| react, react-dom | Builds the UI |
| react-router-dom | Page navigation |
| axios | HTTP calls to the API |
| vite | Dev server and build |

---

## 8. Key Logic

**Repeat reminders:** start at `reminderDate` + `reminderTime`. If `repeatEveryHours` is set, add that many hours for each next reminder. Stop when the next time is after `expiryDate`.

**Reminder job (every minute):**
1. Find documents where `expiry.nextReminderAt <= now`.
2. Update `nextReminderAt` only if its value still matches what was read. This prevents duplicate notifications.
3. Create a notification for the user.

**ICS export:** `GET /api/documents/:id/ics` returns a `text/calendar` file. Use a stable `UID` so re-importing updates the event instead of duplicating it. The user imports it in Google Calendar under Settings → Import & export.

**Auth:** bcrypt hashes passwords. JWT in an HTTP-only cookie. Every query filters by `req.user.id`, so users only see their own data.

---

## 9. API Summary

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/register` | Create account |
| POST | `/api/auth/login` | Login |
| GET | `/api/folders` | Folder tree |
| POST | `/api/folders` | Create folder |
| GET | `/api/documents` | List (filter by folder, category) |
| POST | `/api/documents` | Upload (multipart) |
| GET | `/api/documents/:id` | Details |
| GET | `/api/documents/:id/download` | Download file |
| PATCH | `/api/documents/:id` | Update fields |
| DELETE | `/api/documents/:id` | Delete document and file |
| GET | `/api/documents/:id/ics` | Download calendar event |
| GET | `/api/notifications` | List notifications |
| GET | `/api/notifications/unread-count` | Unread count |
| PATCH | `/api/notifications/:id/read` | Mark as read |

---

## 10. Testing

| Test | Expected result |
|------|-----------------|
| Register with existing email | 409 |
| Access document without login | 401 |
| Insurance upload without expiryDate | 400 |
| financialYear `2025-27` | 400 |
| User B opens User A's document | 404 |
| Repeat 6h from 09:00 | Next reminder 15:00 |
| Reminder processed twice | One notification only |
| `.ics` response | Contains `BEGIN:VCALENDAR` |

Run tests with `cd server && npm test`.

---

## 11. Setup

**Backend**
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
Run with `npm run dev`.

**Frontend**
```bash
cd client
npm install
```
Create `client/.env`:
```
VITE_API_URL=http://localhost:5000/api
```
Run with `npm run dev`, then open `http://localhost:5173`.

---

## 12. Limitations

- Calendar export is one-way. Changed dates need a new `.ics` import.
- Reminders can be up to one minute late.
- Files are stored on local disk. Use cloud storage in production.
- Timezone handling is simplified. Use `luxon` for accuracy.

---

## 13. Future Enhancements

- Google Calendar API sync
- Email reminders with nodemailer
- Cloud storage (S3)
- Document search

---

## 14. License

For learning purposes. Add MIT or another license before sharing publicly.
