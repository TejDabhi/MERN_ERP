# DocumentVault – System Architecture Options

## 1. Project Overview
DocumentVault manages company documents, expiry dates, reminders, and Google Calendar integration.

## 2. Architecture Options

### Option 1: Simple Monolithic Architecture
- Frontend: React
- Backend: Node.js + Express.js
- Database: MongoDB
- Calendar: Google Calendar API
- All backend features are managed in one application.
- **Best for:** Small projects and quick development.

### Option 2: Modular Monolithic Architecture
- Frontend: React
- Backend: Node.js + Express.js
- Database: MongoDB
- Separate modules for documents, reminders, users, and calendar.
- Modules share one backend but maintain separate responsibilities.
- **Best for:** Easy maintenance and future expansion.

### Option 3: Background Job Architecture
- Frontend: React
- Backend: Node.js + Express.js
- Database: MongoDB
- Background jobs check expiry dates and synchronize calendar events.
- Failed calendar operations can be retried automatically.
- **Best for:** Reliable reminders and automatic synchronization.

### Option 4: Microservices Architecture
- Frontend: React
- Separate services for documents, reminders, and calendar integration.
- Database: MongoDB, potentially separated by service.
- Services communicate through APIs or messaging.
- **Best for:** Large systems with independently scalable services.
- **Limitation:** More complex to develop and maintain.

## 3. Common Features
- Add, view, update, and delete document details.
- Set document issue and expiry dates.
- Send reminders before expiry.
- Synchronize expiry events with a shared company Google Calendar.
- Control access based on user roles.
- Track calendar synchronization status and errors.

## 4. Recommendation for Discussion
**Suggested starting point: Option 2 + Option 3.**

Use a modular monolithic backend with background jobs for expiry checking and calendar synchronization. This keeps the system manageable while supporting automatic reminders and retries.

## 5. Senior's Decision
- [ ] Option 1 – Simple Monolithic
- [ ] Option 2 – Modular Monolithic
- [ ] Option 3 – Background Job Architecture
- [ ] Option 2 + Option 3 – Recommended Combination
- [ ] Option 4 – Microservices

**Final architecture:** To be selected after discussion with the senior developer.
