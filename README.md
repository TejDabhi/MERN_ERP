Markdown
# DocumentVault — Engineering Architecture, SDLC Strategy & Implementation Specification

---

## 1. Project Overview & System Architecture

### 1.1 Executive Summary
**DocumentVault** is an enterprise-grade document management and compliance subsystem designed to run within a full-stack MERN application (MongoDB, Express.js, React.js, Node.js). It handles secure ingestion, encrypted storage, and lifecycle tracking of confidential, time-sensitive corporate documents (statutory licenses, tax clearances, vendor contracts, lease agreements, nondisclosure agreements, and corporate approvals).

The system integrates directly with an **E-Locker / Encrypted Storage Vault** for binary object storage and a **Shared Corporate Google Calendar** for automated schedule visibility. It enforces automated state transitions (`ACTIVE` -> `EXPIRING_SOON` -> `EXPIRED`), multi-tiered proactive reminder triggers (pre-expiry), and post-expiry overdue escalation workflows.

### 1.2 High-Level Architecture Topology

+-----------------------------------------------------------------------------------+
|                                 Client Layer                                      |
|                     (React.js SPA with Material-UI / Axios)                       |
+------------------------------------------+----------------------------------------+
|
| HTTPS / TLS 1.3 (JWT + RBAC Bearer)
v
+-----------------------------------------------------------------------------------+
|                        API Gateway / Express Application                          |
|                                                                                   |
|  [Auth Middleware] --> [RBAC / Department Scope] --> [Multer / Input Validation]  |
+-------------------+------------------------------+--------------------------------+
|                              |
v                              v
+-------------------+--------------+ +-------------+--------------------------------+
|  Primary Metadata Store (NoSQL)  | |  Distributed Job Queue (Agenda.js / Cron)   |
|  (MongoDB Replica Set)           | |                                              |
|                                  | |  - Daily Expiry Scanner (00:30 UTC)          |
|  - VaultDocument Collection      | |  - Multi-tier Pre-Expiry Alert Engine        |
|  - VaultAuditLog (Append-Only)   | |  - Post-Expiry Overdue Escalation Engine     |
|  - IntegrationConfig Store       | |  - Exponential Backoff G-Cal Retry Worker    |
+----------------------------------+ +-------------+--------------------------------+
|                              |
| Direct Stream Transfer        | Google Calendar API v3
v                              v
+-------------------+--------------+ +-------------+--------------------------------+
|  E-Locker / Object Storage Vault | |  Shared Company Google Calendar Target       |
|  (AWS S3 / Azure Blob / MinIO)   | |                                              |
|                                  | |  - Pre-Expiry Event Mapping                  |
|  - AES-GCM-256 Envelope Crypto   | |  - Idempotent Event Modification via eventId |
|  - No Direct Public URIs         | |  - Post-Expiry Visual Escalation (Red Alert) |
|  - Ephemeral Decryption Streams  | |  - Automated De-linking on Archival/Renew    |
+----------------------------------+ +----------------------------------------------+


---

## 2. Feature Tree Breakdown

### 1. DocumentVault
* **1.1 Document Management**
  * **Add Document**: Ingests primary metadata (`documentNumber`, `title`, `description`, `category`, `department`, `securityLevel`).
  * **View Documents**: Paginated list and full-text search engine with filters by department, sensitivity tier, validity window, and tags.
  * **Update Document**: Edits mutable metadata while versioning audit fields and maintaining revision history.
  * **Delete Document**: Soft-delete pipeline with retention cooldowns; permanent hard-delete restricted to Super Admins.
  * **Upload Document**: Stream-buffered binary storage into S3-compatible vaults via AES-256 envelope encryption. Direct public download URLs are prohibited; access is strictly brokered through short-lived signed streams.
* **1.2 Document Expiry Management**
  * **Set Expiry Date**: Sets validity dates (`issueDate`, `effectiveDate`, `expiryDate`) with strict invariant validation (expiryDate > issueDate) and a `neverExpires` toggle for perpetual deeds.
  * **View Upcoming Expiries**: Queries categorized temporal buckets (*Expired*, *Critical 0–7 days*, *Urgent 8–30 days*, and *Upcoming 31–90 days*).
  * **Track Expired Documents**: Background state machine transitioning documents from `ACTIVE` -> `EXPIRING_SOON` -> `EXPIRED`.
  * **Set Reminder Preferences**: Per-document overrides for standard system intervals, supporting multi-tier offsets (e.g., 90, 60, 30, 15, 7, 1 day prior).
* **1.3 Google Calendar Integration**
  * **Connect Shared Company Calendar**: Service Account (with domain-wide delegation) or Admin-consented OAuth2 credentials mapped to a dedicated corporate calendar ID.
  * **Create Expiry Events Automatically**: Creates an all-day event upon document creation containing metadata links, policy references, and responsible departments.
  * **Update Events When Expiry Dates Change**: Detects modifications to `expiryDate`, `title`, or `description` and runs an idempotent patch/update using stored `calendarSync.eventId` references.
  * **Delete Linked Events When Required**: Automatically clears calendar entries when a document is soft-deleted, archived, or marked `RENEWED`.
  * **View Synchronization Status**: Real-time observability displaying flags: `SYNCED`, `PENDING`, `FAILED`, and `UNSYNCED`.
  * **Retry Failed Synchronization**: Exponential backoff scheduler backed by a dead-letter queue (DLQ) to handle downstream 429 and 5xx API limits.
* **1.4 Reminder & Post-Expiry Management**
  * **Configure Reminder Days**: Global default offsets with role-based overrides.
  * **Check Upcoming Expiries**: Read-optimized queries on indexed temporal indices.
  * **Run Scheduled Expiry Checks**: Distributed background runners scheduled via cron patterns (default: `00:30 UTC` daily).
  * **Post-Expiry Alerts & Overdue Escalation**: Positive offset triggers (+1, +3, +7, +15 days overdue) alerting department heads and administrators.
  * **Handle Failed Reminder Operations**: Tracks delivery states across multiple channels (In-app notifications, Email/SMTP, Webhooks) with idempotent deduplication keys.
* **1.5 Access Control**
  * **Authenticate Users**: Hardened JWT validation, multi-factor token exchange, and active session invalidation.
  * **Manage User Permissions**: Fine-grained role hierarchy mapping actions (`create`, `read`, `update`, `delete`, `download`, `manage_integrations`, `audit_read`) across business roles.
  * **Restrict Document Access**: Granular Department Isolation (Legal, HR, Executive, Treasury) and Security Clearances (Confidential, Restricted, Top Secret).
  * **Restrict Calendar Configuration to Administrators**: Enforces strict `ADMIN` checks on integration tokens, refresh flows, and calendar ID configurations.
* **1.6 Integration Management**
  * **Configure Google Calendar Access**: Validates cryptographic key pairs (`client_email`, `private_key`) or OAuth token scopes (`https://www.googleapis.com/auth/calendar.events`).
  * **Check Integration Status**: Continuous diagnostic health checks querying the Google Calendar `/users/me/calendarList` endpoint.
  * **Handle API Errors**: Intercepts Google API errors (`Rate Limit Exceeded`, `Resource Gone 410`, `Invalid Grant`), automatically renewing tokens or entering fallback queues.
  * **Secure Integration Credentials**: Credential envelope encryption at rest via environment Master Encryption Keys before persistence to MongoDB.
* **1.7 Audit and Monitoring**
  * **Track Document Changes**: Append-only changelog capturing mutations, previous states, and updated properties along with actor identity and IP address.
  * **Track Calendar Synchronization**: Detailed ledger of sync operations (payload hashes, upstream request IDs, error bodies).
  * **Record Errors**: Centralized structured JSON error logging.
  * **Monitor Failed Jobs**: Operational metrics exposing dead-letter queue states, failed job volumes, and backoff attempts.

---

## 3. Database Schema Blueprint (MongoDB / Mongoose)

### 3.1 Vault Document Collection (`models/VaultDocument.js`)

```javascript
const mongoose = require('mongoose');
const { Schema } = mongoose;

const ReminderOffsetSchema = new Schema({
  offsetDays: { type: Number, required: true }, // Negative: Pre-expiry (-30, -7). 0: Due date. Positive: Overdue (+3, +7).
  type: {
    type: String,
    enum: ['BEFORE_EXPIRY', 'ON_EXPIRY', 'AFTER_EXPIRY'],
    default: 'BEFORE_EXPIRY'
  },
  channel: { 
    type: String, 
    enum: ['IN_APP', 'EMAIL', 'BOTH', 'WEBHOOK'], 
    default: 'BOTH' 
  },
  triggerDate: { type: Date, required: true, index: true },
  isTriggered: { type: Boolean, default: false, index: true },
  triggeredAt: { type: Date },
  failureReason: { type: String }
}, { _id: false });

const VaultDocumentSchema = new Schema({
  documentNumber: { 
    type: String, 
    required: true, 
    unique: true, 
    trim: true, 
    index: true 
  },
  title: { type: String, required: true, trim: true, index: 'text' },
  description: { type: String, trim: true },
  category: {
    type: String,
    required: true,
    enum: [
      'LEGAL_CONTRACT',
      'CORPORATE_STATUTORY',
      'TREASURY_FINANCE',
      'VENDOR_AGREEMENT',
      'COMPLIANCE_CERTIFICATE',
      'INSURANCE_POLICY',
      'EMPLOYMENT_AGREEMENT',
      'INTELLECTUAL_PROPERTY',
      'APPROVALS',
      'PURCHASE_ORDER'
    ],
    index: true
  },
  department: { 
    type: String, 
    required: true, 
    enum: ['LEGAL', 'FINANCE', 'HR', 'OPERATIONS', 'EXECUTIVE', 'ACCOUNTS', 'PURCHASE'],
    index: true 
  },
  securityLevel: {
    type: String,
    required: true,
    enum: ['CONFIDENTIAL', 'RESTRICTED', 'TOP_SECRET'],
    default: 'CONFIDENTIAL',
    index: true
  },
  
  // Temporal Invariants: issueDate <= effectiveDate < expiryDate (unless neverExpires: true)
  issueDate: { type: Date, required: true },
  effectiveDate: { type: Date, required: true },
  expiryDate: { type: Date, default: null, index: true },
  neverExpires: { type: Boolean, default: false },

  // Lifecycle State
  status: {
    type: String,
    enum: ['ACTIVE', 'EXPIRING_SOON', 'EXPIRED', 'ARCHIVED', 'TERMINATED'],
    default: 'ACTIVE',
    index: true
  },

  // Configured Reminders (Multi-tier Pre- and Post-Expiry)
  reminderOffsets: [ReminderOffsetSchema],

  // Google Calendar Integration Mapping
  calendarSync: {
    syncEnabled: { type: Boolean, default: true },
    calendarId: { type: String, default: null },
    eventId: { type: String, default: null, index: true },
    status: { 
      type: String, 
      enum: ['UNSYNCED', 'PENDING', 'SYNCED', 'FAILED'], 
      default: 'UNSYNCED' 
    },
    lastSyncedAt: { type: Date },
    lastAttemptAt: { type: Date },
    retryCount: { type: Number, default: 0 },
    lastErrorMessage: { type: String }
  },

  // E-Locker Object Storage Metadata
  storage: {
    storageProvider: { type: String, default: 'ELocker-S3' },
    storageKey: { type: String, required: true },
    fileName: { type: String, required: true },
    fileMimeType: { type: String, required: true },
    fileSizeBytes: { type: Number, required: true },
    sha256Checksum: { type: String, required: true },
    encryptionAlgorithm: { type: String, default: 'AES-GCM-256' },
    isArchived: { type: Boolean, default: false }
  },

  // Soft Deletion Tracking
  isDeleted: { type: Boolean, default: false, index: true },
  deletedAt: { type: Date, default: null },
  deletedBy: { type: Schema.Types.ObjectId, ref: 'User', default: null },

  // Ownership References
  createdBy: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  updatedBy: { type: Schema.Types.ObjectId, ref: 'User', required: true }
}, {
  timestamps: true,
  optimisticConcurrency: true
});

// Compound Indexes for Performance
VaultDocumentSchema.index({ status: 1, expiryDate: 1, isDeleted: 1 });
VaultDocumentSchema.index({ 'calendarSync.status': 1, 'calendarSync.syncEnabled': 1 });
VaultDocumentSchema.index({ department: 1, securityLevel: 1, isDeleted: 1 });
VaultDocumentSchema.index({ 'reminderOffsets.isTriggered': 1, 'reminderOffsets.triggerDate': 1 });

module.exports = mongoose.model('VaultDocument', VaultDocumentSchema);
3.2 Audit Log Collection (models/VaultAuditLog.js)
JavaScript
const mongoose = require('mongoose');
const { Schema } = mongoose;

const VaultAuditLogSchema = new Schema({
  documentId: { type: Schema.Types.ObjectId, ref: 'VaultDocument', required: true, index: true },
  documentNumber: { type: String, required: true },
  action: {
    type: String,
    required: true,
    enum: [
      'DOCUMENT_CREATED',
      'DOCUMENT_ACCESSED',
      'DOCUMENT_METADATA_UPDATED',
      'EXPIRY_DATE_ALTERED',
      'FILE_DOWNLOADED',
      'DOCUMENT_SOFT_DELETED',
      'DOCUMENT_PERMANENT_DELETED',
      'CALENDAR_EVENT_CREATED',
      'CALENDAR_EVENT_UPDATED',
      'CALENDAR_EVENT_DELETED',
      'CALENDAR_SYNC_FAILED',
      'CALENDAR_EVENT_EXPIRED_ALERT',
      'REMINDER_DISPATCHED',
      'OVERDUE_ALERT_DISPATCHED'
    ],
    index: true
  },
  actor: {
    userId: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    email: { type: String, required: true },
    ipAddress: { type: String, required: true },
    userAgent: { type: String }
  },
  diff: {
    previousState: { type: Schema.Types.Mixed },
    newState: { type: Schema.Types.Mixed }
  },
  context: { type: Schema.Types.Mixed }
}, {
  timestamps: { createdAt: true, updatedAt: false }
});

VaultAuditLogSchema.index({ createdAt: -1 });

module.exports = mongoose.model('VaultAuditLog', VaultAuditLogSchema);
3.3 Integration Configuration Collection (models/IntegrationConfig.js)
JavaScript
const mongoose = require('mongoose');
const { Schema } = mongoose;

const IntegrationConfigSchema = new Schema({
  provider: { type: String, required: true, unique: true, default: 'GOOGLE_CALENDAR' },
  isEnabled: { type: Boolean, default: false },
  sharedCalendarId: { type: String, required: true },
  
  // Credentials encrypted with AES-256-GCM using Master System Key
  credentials: {
    authType: { type: String, enum: ['SERVICE_ACCOUNT', 'OAUTH2'], default: 'SERVICE_ACCOUNT' },
    encryptedClientEmail: { type: String, required: true },
    encryptedPrivateKey: { type: String, required: true },
    iv: { type: String, required: true },
    authTag: { type: String, required: true }
  },
  
  operationalState: {
    status: { type: String, enum: ['OPERATIONAL', 'DEGRADED', 'AUTHENTICATION_FAILED'], default: 'OPERATIONAL' },
    lastHealthCheckAt: { type: Date },
    consecutiveFailures: { type: Number, default: 0 },
    lastErrorMessage: { type: String }
  },
  
  updatedBy: { type: Schema.Types.ObjectId, ref: 'User', required: true }
}, {
  timestamps: true
});

module.exports = mongoose.model('IntegrationConfig', IntegrationConfigSchema);
4. Role-Based Access Control (RBAC) Specification
Action / Capability	Viewer	Auditor	Department Officer	Vault Admin	Super Admin
View Metadata	Own Dept	All (Read Only)	Own Dept	All	All
Download Document Stream	With Approval	Restrictive	Own Dept	All	All
Upload / Add Document	❌	❌	Allowed	Allowed	Allowed
Modify Document / Expiry	❌	❌	Allowed	Allowed	Allowed
Soft Delete Document	❌	❌	❌	Allowed	Allowed
Permanent Purge	❌	❌	❌	❌	Allowed
Read Audit Logs	❌	Allowed	❌	Allowed	Allowed
Manage G-Cal Integration	❌	❌	❌	Allowed	Allowed
Force Retry Sync Jobs	❌	❌	❌	Allowed	Allowed
5. Google Calendar Synchronization Engine
5.1 Pre-Expiry & Post-Expiry Event Lifecycle Strategy
Pre-Expiry Creation: When an expiring document is registered, an all-day event is scheduled on the shared calendar with the default palette.

Date Shifts: Modifying the document's expiryDate immediately invokes a PATCH request using the stored eventId.

Event Deletion: Soft deleting, archiving, or marking a document as renewed deletes the linked event.

Post-Expiry Transformation: When a document passes its expiry date:

Title is updated to: [EXPIRED] <Document Title> (<Document Number>).

Description is updated with days overdue, compliance impact, and access URLs.

colorId is shifted to Google Calendar color "11" (Bold Red).

Transparency is shifted from transparent (Free) to opaque (Busy).

5.2 Google Calendar Integration Wrapper (services/googleCalendarService.js)
JavaScript
const { google } = require('googleapis');
const crypto = require('crypto');
const IntegrationConfig = require('../models/IntegrationConfig');

class GoogleCalendarService {
  constructor() {
    this.jwtClient = null;
    this.cachedCalendarId = null;
  }

  decryptString(encryptedData, ivHex, authTagHex, secretKeyHex) {
    const key = Buffer.from(secretKeyHex, 'hex');
    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');
    const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
    decipher.setAuthTag(authTag);
    let decrypted = decipher.update(encryptedData, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
  }

  async getClient() {
    const config = await IntegrationConfig.findOne({ provider: 'GOOGLE_CALENDAR', isEnabled: true });
    if (!config) {
      throw new Error('Google Calendar integration is not configured or disabled.');
    }

    const privateKey = this.decryptString(
      config.credentials.encryptedPrivateKey,
      config.credentials.iv,
      config.credentials.authTag,
      process.env.VAULT_INTEGRATION_MASTER_KEY
    );

    const clientEmail = this.decryptString(
      config.credentials.encryptedClientEmail,
      config.credentials.iv,
      config.credentials.authTag,
      process.env.VAULT_INTEGRATION_MASTER_KEY
    );

    this.cachedCalendarId = config.sharedCalendarId;
    
    this.jwtClient = new google.auth.JWT(
      clientEmail,
      null,
      privateKey,
      ['[https://www.googleapis.com/auth/calendar.events](https://www.googleapis.com/auth/calendar.events)']
    );

    await this.jwtClient.authorize();
    return google.calendar({ version: 'v3', auth: this.jwtClient });
  }

  buildEventPayload(doc) {
    const dateFormatted = doc.expiryDate.toISOString().split('T')[0];
    return {
      summary: `[EXPIRY] ${doc.title} (${doc.documentNumber})`,
      description: `CONFIDENTIAL TIME-SENSITIVE DOCUMENT\n` +
                   `Doc Number: ${doc.documentNumber}\n` +
                   `Category: ${doc.category}\n` +
                   `Department: ${doc.department}\n` +
                   `Security Clearance: ${doc.securityLevel}\n` +
                   `Vault ID: ${doc._id}\n\n` +
                   `Managed automatically via DocumentVault.`,
      start: { date: dateFormatted },
      end: { date: dateFormatted },
      transparency: 'transparent',
      extendedProperties: {
        private: {
          vaultDocumentId: doc._id.toString(),
          documentNumber: doc.documentNumber
        }
      }
    };
  }

  async createOrUpdateEvent(doc) {
    const calendar = await this.getClient();
    const eventPayload = this.buildEventPayload(doc);

    if (doc.calendarSync && doc.calendarSync.eventId) {
      try {
        const response = await calendar.events.update({
          calendarId: this.cachedCalendarId,
          eventId: doc.calendarSync.eventId,
          requestBody: eventPayload
        });
        return { eventId: response.data.id, status: 'SYNCED' };
      } catch (err) {
        if (err.code !== 404 && err.code !== 410) throw err;
      }
    }

    const response = await calendar.events.insert({
      calendarId: this.cachedCalendarId,
      requestBody: eventPayload
    });

    return { eventId: response.data.id, status: 'SYNCED' };
  }

  async markEventAsExpired(doc, daysOverdue = 0) {
    const calendar = await this.getClient();
    if (!doc.calendarSync || !doc.calendarSync.eventId) return;

    const dateFormatted = doc.expiryDate.toISOString().split('T')[0];
    const updatedPayload = {
      summary: `[EXPIRED] ${doc.title} (${doc.documentNumber})`,
      description: `CRITICAL ATTENTION: THIS DOCUMENT EXPIRED ON ${dateFormatted} (${daysOverdue} days overdue).\n\n` +
                   `Doc Number: ${doc.documentNumber}\n` +
                   `Department: ${doc.department}\n` +
                   `Status: EXPIRED - IMMEDIATE ACTION REQUIRED\n` +
                   `Access Record: ${process.env.APP_BASE_URL}/vault/documents/${doc._id}`,
      start: { date: dateFormatted },
      end: { date: dateFormatted },
      colorId: '11', // Bold Red
      transparency: 'opaque'
    };

    await calendar.events.patch({
      calendarId: this.cachedCalendarId,
      eventId: doc.calendarSync.eventId,
      requestBody: updatedPayload
    });
  }

  async deleteEvent(calendarId, eventId) {
    if (!eventId) return;
    const calendar = await this.getClient();
    try {
      await calendar.events.delete({
        calendarId: calendarId || this.cachedCalendarId,
        eventId: eventId
      });
    } catch (err) {
      if (err.code !== 404 && err.code !== 410) throw err;
    }
  }
}

module.exports = new GoogleCalendarService();
6. Background Scheduling & Job Definitions (jobs/vaultJobs.js)
JavaScript
const VaultDocument = require('../models/VaultDocument');
const VaultAuditLog = require('../models/VaultAuditLog');
const GoogleCalendarService = require('../services/googleCalendarService');

module.exports = function(agenda) {

  // Job 1: Daily Expiration Evaluator & Reminder Processor (Runs 00:30 UTC)
  agenda.define('EVALUATE_EXPIRIES_AND_REMINDERS', { lockLifetime: 1000 * 60 * 5 }, async (job, done) => {
    const now = new Date();

    try {
      // 1. Transition ACTIVE -> EXPIRING_SOON (<= 30 Days)
      const thirtyDays = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);
      await VaultDocument.updateMany(
        { isDeleted: false, neverExpires: false, expiryDate: { $lte: thirtyDays, $gt: now }, status: 'ACTIVE' },
        { $set: { status: 'EXPIRING_SOON' } }
      );

      // 2. Identify Newly Expired Documents (expiryDate < now and not yet marked EXPIRED)
      const newlyExpiredDocs = await VaultDocument.find({
        isDeleted: false,
        neverExpires: false,
        expiryDate: { $lte: now },
        status: { $in: ['ACTIVE', 'EXPIRING_SOON'] }
      });

      for (const doc of newlyExpiredDocs) {
        doc.status = 'EXPIRED';
        await doc.save();

        // Escalate Google Calendar Entry to Red
        await GoogleCalendarService.markEventAsExpired(doc, 0);

        await VaultAuditLog.create({
          documentId: doc._id,
          documentNumber: doc.documentNumber,
          action: 'CALENDAR_EVENT_EXPIRED_ALERT',
          actor: { userId: doc.updatedBy, email: 'system.engine@internal', ipAddress: '127.0.0.1' },
          context: { status: 'EXPIRED', markedAt: new Date() }
        });
      }

      // 3. Process Reminders (Pre-Expiry and Post-Expiry Overdue)
      const documentsNeedingReminders = await VaultDocument.find({
        isDeleted: false,
        'reminderOffsets.isTriggered': false,
        'reminderOffsets.triggerDate': { $lte: now }
      });

      for (const doc of documentsNeedingReminders) {
        let docModified = false;
        const daysOverdue = Math.floor((now - new Date(doc.expiryDate)) / (1000 * 60 * 60 * 24));

        for (const reminder of doc.reminderOffsets) {
          if (!reminder.isTriggered && reminder.triggerDate <= now) {
            try {
              if (reminder.type === 'AFTER_EXPIRY') {
                await GoogleCalendarService.markEventAsExpired(doc, daysOverdue);
              }

              reminder.isTriggered = true;
              reminder.triggeredAt = new Date();
              docModified = true;

              await VaultAuditLog.create({
                documentId: doc._id,
                documentNumber: doc.documentNumber,
                action: reminder.type === 'AFTER_EXPIRY' ? 'OVERDUE_ALERT_DISPATCHED' : 'REMINDER_DISPATCHED',
                actor: { userId: doc.updatedBy, email: 'system.engine@internal', ipAddress: '127.0.0.1' },
                context: { offsetDays: reminder.offsetDays, type: reminder.type, channel: reminder.channel }
              });
            } catch (dispatchErr) {
              reminder.failureReason = dispatchErr.message;
              docModified = true;
            }
          }
        }
        if (docModified) await doc.save();
      }

      done();
    } catch (err) {
      done(err);
    }
  });

  // Job 2: Asynchronous Resilient Calendar Sync (Worker Task)
  agenda.define('SYNC_SINGLE_CALENDAR_EVENT', { retryFactor: 2, maxRetries: 5 }, async (job, done) => {
    const { documentId } = job.attrs.data;
    const doc = await VaultDocument.findById(documentId);

    if (!doc || doc.isDeleted || !doc.calendarSync.syncEnabled || doc.neverExpires) {
      return done();
    }

    try {
      doc.calendarSync.status = 'PENDING';
      doc.calendarSync.lastAttemptAt = new Date();

      const { eventId, status } = await GoogleCalendarService.createOrUpdateEvent(doc);

      doc.calendarSync.eventId = eventId;
      doc.calendarSync.status = status;
      doc.calendarSync.lastSyncedAt = new Date();
      doc.calendarSync.retryCount = 0;
      doc.calendarSync.lastErrorMessage = null;
      await doc.save();

      done();
    } catch (err) {
      doc.calendarSync.status = 'FAILED';
      doc.calendarSync.retryCount += 1;
      doc.calendarSync.lastErrorMessage = err.message;
      await doc.save();
      done(err);
    }
  });
};
7. Software Development Life Cycle (SDLC) Strategy
Phase 1: Requirements Analysis
   └── Functional specifications, RBAC mapping, expiry invariants, post-expiry escalation rules.
Phase 2: Architectural & Schema Design
   └── Mongoose schema design, compound indices, E-Locker crypto design, Agenda queue workflows.
Phase 3: Core Implementation
   └── CRUD controllers, Multer streaming, Google Calendar wrapper, background workers, React UI.
Phase 4: Validation & Quality Assurance
   └── Invariant tests, mock API rate limits (HTTP 429), duplicate check tests, RBAC boundary tests.
Phase 5: Production Deployment
   └── Containerization, secret injection (KMS), Agenda scheduler initialization, smoke testing.
Phase 6: Maintenance & Operations
   └── Sync monitoring, audit log review, dead-letter queue analysis, quarterly credential rotation.
Development Order
Requirements & Security Approval: Finalize schema fields, department scopes, and notification intervals.

Database & Core Engine: Implement Mongoose schemas, compound indices, and audit log hooks.

Storage Engine Layer: Configure Multer memory streaming and the E-Locker abstraction.

Calendar Integration: Implement the GoogleCalendarService wrapper and OAuth/Service Account decryptor.

Scheduler & Cron Runners: Configure Agenda.js jobs for daily expiry evaluation, pre-expiry reminders, and post-expiry escalations.

Frontend Development: Build React components, state hooks, search filters, and the sync retry widget.

End-to-End Verification: Validate calendar patching, duplicate event prevention, and invariant enforcement.

Deployment & CI/CD: Build production container images and configure environment secrets.

8. Definition of Completion (DoD)
A release candidate is deemed complete and production-ready only when all of the following criteria are validated:

[ ] Access Isolation: Department-level filtering prevents unauthorized cross-departmental record access.

[ ] Temporal Validation: Invariants are enforced (expiryDate > issueDate); perpetual documents bypass expiry checks.

[ ] Pre-Expiry Sync: Active documents synchronize to the shared Google Calendar without blocking primary API response times.

[ ] Idempotent Updates: Rescheduled expiry dates patch the existing Google Calendar entry (eventId) without creating duplicate events.

[ ] Post-Expiry Escalation: Expired documents transition to EXPIRED, their Google Calendar events turn red (colorId: "11"), and overdue notifications trigger on schedule.

[ ] De-linking on Removal: Soft-deleting, renewing, or archiving a document cleanly removes its linked event from the Google Calendar.

[ ] Fault Resilience: Google API 429 and 5xx errors trigger exponential backoff without stalling application execution.

[ ] Encrypted Storage: Binary documents in the E-Locker are stored with AES-256 envelope encryption, with access provided only via ephemeral decrypted streams.

[ ] Audit Completeness: All record mutations, viewings, file downloads, and synchronization attempts are logged to VaultAuditLog.

9. Developer Q&A Reference Guide
Q1: How does the system prevent duplicate events on Google Calendar during updates or sync retries?
Answer: The VaultDocument schema stores the external calendarSync.eventId returned by Google APIs. When updates occur or a retry job executes, the integration checks for this ID. If present, it executes an idempotent calendar.events.update or patch call rather than insert.

Q2: Why are positive numbers used in the reminder offsets array?
Answer: Negative offsets (e.g., -30, -7) represent triggers scheduled before expiration. Positive offsets (e.g., +1, +7) represent overdue escalation reminders executed after the expiration date has passed.

Q3: What happens to the Google Calendar event if a document is deleted?
Answer: The delete controller marks the database record as isDeleted: true and triggers a background task that calls GoogleCalendarService.deleteEvent(calendarId, eventId). This removes the event from the shared calendar so no stale alerts remain.

Q4: How is sensitive data handled during Google Calendar synchronization?
Answer: Only non-sensitive operational metadata is synchronized (Document Number, Title, Department, Validity End Date, and an internal portal reference link). Document binaries, raw contents, and confidential terms are never transmitted to external calendar APIs.

Q5: How does the E-Locker handle file security during downloads?
Answer: Files stored in the E-Locker are encrypted using unique data keys wrapped by a master key. The download endpoint verifies JWT claims and departmental clearances, decrypts the binary stream in-memory, and pipes it directly to the response with Content-Disposition: attachment. Public static URLs are never generated.
