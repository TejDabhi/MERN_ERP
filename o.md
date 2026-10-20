# DocumentVault Demo (HTML)

A single-file, browser-only demo of the DocumentVault document locker. It has no login, no server, and no database. Open `documentvault-demo.html` in any modern browser or HTML runner.

This demo shows the user interface and business rules. The full MERN version is described in `README.md`.

---

## 1. What the Demo Shows

- Two fixed default folders: **Personal** and **Company**
- A category tree with radio buttons (category → subcategory)
- Upload form with file name, file, description, and issue date
- Expiry and reminder fields for expiry subcategories (Insurance, Agreement)
- Financial year field for financial-year subcategories (Income Tax Return, Accounts)
- Automatic reminder date, one month before expiry
- Repeat reminders every 3, 6, or 12 hours
- Document list with view, download, `.ics` export, and delete
- Notification bell with unread count and mark-as-read

---

## 2. Folders

| Folder | Created by | Can user add, rename, or delete? |
|--------|-----------|----------------------------------|
| Personal | Demo seed data | No |
| Company | Demo seed data | No |

- Documents are saved into one of these two folders, chosen in the **Folder** dropdown.
- The "Add folder" form is not in this demo. The demo does not support subfolders.
- Clicking a folder in the left panel filters the document list to that folder.

---

## 3. Categories and Subcategories

The category tree comes from `CATEGORY_TREE` in the script. The tree shown depends on the folder selected in the **Folder** dropdown. Selecting **Personal** shows the Personal subcategories, and selecting **Company** shows the Company subcategories.

**Personal**

| Subcategory | Extra fields |
|-------------|--------------|
| Passport Size Photos | None |
| PAN Card | None |
| Driving Licence | None |
| Insurance | Expiry fields |
| Education | None |

**Company**

| Subcategory | Extra fields |
|-------------|--------------|
| Documents For QR | None |
| GST Registration | None |
| PF Certificate | None |
| DSC | None |
| Agreement | Expiry fields |
| Income Tax Return | Financial year |
| Accounts | Financial year |

Each subcategory has flags in the code: `expiry: true` shows the expiry section, and `fy: true` shows the financial year field.

---

## 4. Upload Form Rules

| Field | Required | Rule |
|-------|----------|------|
| File name | Yes | Cannot be empty |
| File | Yes | One file must be chosen |
| Description | No | Free text |
| Issue date | Yes | Defaults to today, can be changed |
| Folder | Yes | Personal or Company |
| Subcategory | Yes | One radio button must be selected |
| Expiry date | Expiry subcategories only | Must be at least one month after issue date |
| Reminder time | Expiry subcategories only | Defaults to 09:00 |
| Repeat every | No | Once only, 3, 6, or 12 hours |
| Financial year | Financial-year subcategories only | Format `YYYY-YY`, for example `2025-26` |

**Minimum expiry date:** the issue date plus one month. Example: issue `2026-10-20` gives a minimum expiry of `2026-11-20`.

---

## 5. Reminder Logic

**Reminder date** is calculated from the expiry date:

- Take the expiry date and go back one month.
- If the earlier month is shorter, use its last day. Example: expiry `2027-03-31` gives reminder `2027-02-28`.
- If the result is already in the past, use today.

Example: expiry `2026-12-15` gives reminder `2026-11-15`.

**Repeat reminders:** the first reminder fires at `reminderDate` + `reminderTime`. If a repeat interval is set, the next reminder is scheduled that many hours later. Repeats stop after the expiry date.

**Reminder checker:** a timer runs every 10 seconds and checks each document's `nextReminderAt`. This matches the server cron job in the full project.

**Duplicate prevention:** `nextReminderAt` moves forward before the notification is created. A reminder is never sent twice for the same time.

---

## 6. Document List

| Column | Content |
|--------|---------|
| File name | Name entered by the user |
| Category | Category and subcategory |
| Issued | Issue date |
| Details | Description, reminder schedule, and financial year when present |
| Expiry | Expiry date, days left (30 or fewer), or Expired |
| Actions | View, Download, .ics (expiry documents only), Delete |

The **Category** filter above the list shows Personal or Company documents only.

---

## 7. ICS Export

Clicking **.ics** downloads a calendar file for an expiry document.

- Event starts at the expiry date and reminder time, and lasts 30 minutes.
- The file uses a stable `UID` in the form `doc-<id>@documentvault`.
- Import it in Google Calendar under **Settings → Import & export**.
- Import is one-way. If the expiry date changes, download and import a new file.

---

## 8. Notifications

- The bell icon shows the unread count.
- Each due reminder creates a notification with the file name and expiry date.
- Each notification can be marked as read, or all can be marked as read at once.
- The bell opens and closes the notification panel.

---

## 9. Data and Storage

- All data is stored in JavaScript memory.
- Refreshing the page clears all documents, notifications, and folder changes.
- Uploaded files are kept as browser file objects. They are not saved to disk.
- Viewing and downloading use temporary browser URLs.

---

## 10. Known Limitations

- No login, no user accounts, and no server.
- Reminders only fire while the page is open.
- The category tree depends on the folder name matching a key in `CATEGORY_TREE`. Subfolders would show no categories, but the demo does not create subfolders.
- Date limits are enforced in the form and the script, but this is not a security boundary. The full MERN version must enforce them on the server.
- Reminder timing has no timezone handling. Times use the browser's local timezone.

---

## 11. How to Run

1. Download `documentvault-demo.html`.
2. Open it in Chrome, Edge, or Firefox by double-clicking, or paste it into an online HTML runner.
3. Choose a folder, pick a subcategory, fill in the form, and click **Save Document**.

No installation is needed.
