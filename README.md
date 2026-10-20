flowchart TD
    %% Styling Definitions
    classDef route fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#0d47a1;
    classDef mid fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1.5px,color:#4a148c;
    classDef ctrl fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#bf360c;
    classDef srv fill:#e8f5e9,stroke:#2e7d32,stroke-width:1.5px,color:#1b5e20;
    classDef db fill:#fbe9e7,stroke:#d84315,stroke-width:2px,color:#bf360c;
    classDef cron fill:#ede7f6,stroke:#512da8,stroke-width:2px,color:#311b92;

    %% 1. Request & Middlewares
    Client([HTTP Request /api/vault/*]):::route --> Routes[routes.js]:::route
    
    subgraph Middlewares ["Middlewares & Request Processing"]
        Routes --> RBAC[vaultRbac.js<br/>Auth & Department Scope]:::mid
        RBAC --> Multer[vaultMulterConfig.js<br/>Multipart File Parsing]:::mid
        Multer --> SigCheck[fileSignatureValidator.js<br/>MIME & Magic Number Check]:::mid
        SigCheck --> Validator[vaultValidator.js<br/>expiryDate > issueDate Check]:::mid
    end

    %% 2. Controllers Routing
    Validator -->|Upload / Edit / Delete| C_Vault[vaultController.js]:::ctrl
    Validator -->|Secure Download Request| C_Stream[streamController.js]:::ctrl
    Validator -->|Configure Alert Rules| C_Remind[reminderController.js]:::ctrl
    Validator -->|Manual Calendar Retry| C_Sync[calendarSyncController.js]:::ctrl
    Validator -->|Audit Logs Query| C_Audit[auditLogController.js]:::ctrl

    %% 3. Ingestion Flow (vaultController)
    subgraph Ingestion ["Ingestion & Storage Flow"]
        C_Vault --> OffsetCalc[reminderOffsetCalculator.js<br/>Compute Trigger Dates]:::srv
        C_Vault --> Storage[fileStorageService.js<br/>Save Encrypted File]:::srv
        Storage --> Crypto[cryptoService.js<br/>AES-256-GCM Encryption]:::srv
        Storage --> VaultDisk[(Vault Storage Directory)]:::db
    end

    %% 4. Ephemeral Download Flow
    subgraph Download ["Ephemeral Streaming Pipeline"]
        C_Stream --> Storage
        Storage -->|Read Raw Encrypted Stream| Crypto
        Crypto -->|Stream Decrypted Buffer| ResStream([HTTP Response Stream]):::route
    end

    %% 5. Database Persistence
    C_Vault --> M_Doc[(VaultDocument.js)]:::db
    C_Vault --> M_Audit[(VaultAuditLog.js)]:::db
    C_Stream --> M_Audit
    C_Audit --> M_Audit

    %% 6. Google Calendar Sync
    subgraph CalendarSync ["Google Calendar Synchronization"]
        C_Vault --> GCal[googleCalendarService.js]:::srv
        C_Sync --> GCal
        GCal --> M_Integ[(IntegrationConfig.js)]:::db
        GCal --> EvtBuilder[calendarEventBuilder.js<br/>Construct Event Payload]:::srv
        GCal --> GoogleAPI([Google Calendar API v3]):::route
        GoogleAPI -->|Return eventId & Status| M_Doc
    end

    %% 7. Background Jobs / Cron Engines
    subgraph BackgroundWorkers ["Cron Engine & Asynchronous Workers"]
        Cron_Daily[vaultJobs.js<br/>Runs Daily at 00:30 UTC]:::cron
        Cron_Retry[calendarRetryWorker.js<br/>Exponential Backoff Retries]:::cron

        Cron_Daily -->|1. Transition Status<br/>ACTIVE to EXPIRING_SOON to EXPIRED| M_Doc
        Cron_Daily -->|2. Escalate Overdue Events<br/>Turn Red colorId 11| GCal
        Cron_Daily -->|3. Evaluate Trigger Dates| Notif[notificationService.js]:::srv
        
        Notif --> InApp([In-App Notification]):::route
        Notif --> Email([Email Dispatcher]):::route
        
        Cron_Retry -->|Retry Failed Events| GCal
        Cron_Daily --> M_Audit
        Cron_Retry --> M_Audit
    end
