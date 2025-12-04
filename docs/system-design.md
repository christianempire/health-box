# 1. Overview

## 1.1 Goal

Build an MVP web platform that lets users **organize and use their medical information primarily through Google services**, with:

* **Angular** SPA as the frontend.
* **NestJS** backend as a thin API + orchestration layer.
* **Google services** as main storage and identity:

  * Google OAuth (Sign in with Google)
  * Google Drive (documents, images, PDFs)
  * Google Calendar (appointments, exams)
* A **minimal database** only for indexing and business state (status, metadata, references to Drive & Calendar).

The platform is essentially a **health dashboard on top of the user’s own Google account**.

## 1.2 Non-goals (for MVP)

* No native mobile apps (web only).
* No integration with clinics / labs / hospitals APIs.
* No two-way sync from Calendar/Drive changes back into the app (beyond basic error handling).
* No OCR or full-text search inside PDFs/images.
* No WebAuthn/biometric login in v1.

---

# 2. High-level Architecture

## 2.1 Components

1. **Frontend (Angular)**

   * SPA with:

     * Login via Google.
     * “About You” / Health Profile wizard.
     * “Boxes” for Exams, Prescriptions, Appointments, Vaccines, etc.
     * File uploads & listings.
     * Buttons to add/update Calendar events.

2. **Backend (NestJS)**

   * REST API consumed by Angular.
   * Handles:

     * Google OAuth token verification.
     * User provisioning.
     * Google Drive & Calendar API calls.
     * Minimal DB persistence & querying.
   * Modular architecture with Nest modules:

     * `AuthModule`
     * `UsersModule`
     * `GoogleModule`
     * `DriveModule`
     * `CalendarModule`
     * `HealthProfileModule`
     * `DocumentsModule`
     * `ExamsModule`
     * `AppointmentsModule`
     * `VaccinesModule`
     * (Optional later) `SyncModule` for Drive reconciliation jobs.

3. **Database**

   * Relational (PostgreSQL or MySQL recommended).
   * Used as an **index & state store**, not file storage.
   * Stores:

     * Users & Google IDs
     * Drive file metadata (ID, folder, type, status)
     * Calendar event IDs & status
     * Health profile structured data
     * Consents and audit fields

4. **Google Services**

   * **Google Identity / OAuth2**: user authentication.
   * **Google Drive**: user-owned health documents (PDFs, images, etc.).
   * **Google Calendar**: appointments & exams events.

## 2.2 Data Flow (typical session)

1. User visits Angular app → clicks “Sign in with Google”.
2. Frontend uses Google JS SDK / OAuth 2.0 → obtains ID token.
3. Frontend sends ID token to NestJS `/auth/google` endpoint.
4. Backend verifies token with Google, extracts `sub` (Google user ID), email, name.
5. Backend creates/updates user in DB; returns app JWT + basic profile.
6. On first login:

   * Backend ensures Drive folder structure is created in user’s Drive.
   * Creates initial records in DB as needed.
7. User actions (upload exam, create appointment, etc.) trigger:

   * Drive API calls (upload file, set properties).
   * Calendar API calls (create event).
   * DB updates (index & state).

---

# 3. Google OAuth & Security Design

## 3.1 Authentication Flow

* **Only Google login** (no local password for MVP).
* Scopes (minimally):

  * `openid email profile`
  * `https://www.googleapis.com/auth/drive.file`
    (App can create/access files it creates or that the user selects.)
  * `https://www.googleapis.com/auth/calendar.events`

### Backend Steps

1. Receive ID token from Angular.
2. Verify with Google (via `google-auth-library`).
3. Extract:

   * `sub` (Google user ID)
   * `email`
   * `name`
   * `picture` (optional)
4. Upsert into `users` table.
5. Generate **app JWT** with:

   * `userId` (internal ID)
   * `googleUserId`
   * `roles` (for future admin features).

The app JWT is then used for all API calls.

## 3.2 Token Storage

* **In DB**:

  * Store encrypted Google **refresh token** (if using server-side OAuth flow).
  * Store last known access token expiration time if needed.
* **In frontend**:

  * Store only the app JWT in `localStorage` or secure cookie (depending on threat model).
  * Do not store Google tokens in frontend beyond immediate use.

## 3.3 Authorization

* All API routes protected by NestJS guards:

  * `JwtAuthGuard` → checks app JWT.
* Access control:

  * Each resource is associated with `user_id`.
  * Standard “user can only read/write their own records”.

---

# 4. Google Drive Design

## 4.1 Folder Structure

For each user, we create a root folder:

* Root:
  `HealthVault (AppName)`

  * `/Profile`
  * `/Exams`
  * `/Appointments`
  * `/Prescriptions`
  * `/Vaccines`
  * `/MentalHealth` (if used)
  * `/Other` (optional generic box)

Everything we create/write goes under this hierarchy.

### Naming conventions

* Root folder name example: `HealthVault - {UserNameOrEmail}`
* Files:

  * Exams: `Exam - {Type} - {YYYY-MM-DD}`
  * Appointments: `Appointment - {DoctorOrSpecialty} - {YYYY-MM-DD}`
  * Prescriptions: `Prescription - {DoctorOrMedication} - {YYYY-MM-DD}`
  * Vaccines: `Vaccine - {Name} - {YYYY-MM-DD}`

## 4.2 Use of File Properties

Each Drive file we create gets `appProperties` for easy identification:

Example `appProperties`:

```json
{
  "appType": "healthvault",        // constant to find "our" files
  "boxType": "exam",               // exam | appointment | prescription | vaccine | profile | other
  "status": "pending",             // pending | scheduled | completed | expired etc.
  "category": "blood_test",        // optional subtype
  "userId": "internal-user-id",    // link back to our DB user
  "createdByApp": "true"
}
```

We still keep a **DB index**, but `appProperties` help if we need to re-scan Drive.

## 4.3 File Operations

* **Create folder structure**:

  * On first login, backend checks if root folder exists (search by name + property).
  * If not, create root + subfolders.
  * Store the root folder ID in `users` table.

* **Upload file**:

  * Angular → sends file to backend or directly to Drive (for MVP simpler: through backend).
  * Backend:

    * Chooses folder based on box (`/Exams`, `/Vaccines`, etc.).
    * Calls Drive API `files.create` with:

      * `parents: [folderId]`
      * `appProperties` as above.
    * Stores returned `fileId` and metadata in DB.

* **List files per box**:

  * Backend queries **DB index** (not Drive) to get list efficiently.
  * Angular fetches from backend, receives list with titles, dates, status, plus link to open in Drive.

* **Open file from UI**:

  * Angular uses `https://drive.google.com/file/d/{fileId}/view` or similar.

---

# 5. Google Calendar Design

## 5.1 Event Types

We create events for:

* **Appointments** (primary use case)
* Optionally for **Exams** when they are scheduled.

### Event Fields

* `summary`:

  * Appointment: `"Appointment – {DoctorName} ({Specialty})"`
  * Exam: `"Exam – {ExamType}"`

* `description`:

  * Link to relevant Drive doc (`https://drive.google.com/file/d/{fileId}/view`).
  * Notes entered by the user.

* `start` / `end`:

  * Use user’s timezone (stored in `users.timezone`).

* `reminders`:

  * Use default reminders, or custom (e.g., 24h before, 2h before).

## 5.2 Event Management

* Creating appointment/exam:

  * Backend creates event via Calendar API.
  * Stores `calendar_event_id` in DB.

* Updating appointment/exam:

  * Backend updates event by ID.

* Canceling:

  * Backend deletes or updates event with `status: "cancelled"` and marks DB record as cancelled.

> For MVP, we **do not** automatically sync changes done directly in Google Calendar back to our app.

---

# 6. Minimal Database Schema

Relational schema sketch (PostgreSQL-like).

## 6.1 Users

```sql
CREATE TABLE users (
    id              SERIAL PRIMARY KEY,
    google_user_id  VARCHAR(255) UNIQUE NOT NULL,
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255),
    picture_url     TEXT,
    timezone        VARCHAR(64) DEFAULT 'America/Lima',
    drive_root_id   VARCHAR(255),   -- HealthVault root folder ID
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## 6.2 Google Tokens

```sql
CREATE TABLE user_google_tokens (
    user_id         INTEGER PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    refresh_token   TEXT NOT NULL,        -- encrypted at rest
    access_token    TEXT,                 -- optional (can be cached)
    access_expires_at TIMESTAMP,          -- optional
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
```

*(If you use purely backend OAuth with refresh tokens. If you go “front-end auth only” with short-lived tokens, you may adjust this.)*

## 6.3 Health Profile

You can either go big and normalized or keep MVP-friendly and simpler. For MVP, a **wide table** is acceptable:

```sql
CREATE TABLE health_profiles (
    id                  SERIAL PRIMARY KEY,
    user_id             INTEGER UNIQUE REFERENCES users(id) ON DELETE CASCADE,

    -- Basic
    dob                 DATE,
    gender              VARCHAR(32),
    height_cm           NUMERIC(5,2),
    weight_kg           NUMERIC(5,2),

    -- Examples of fields from "About you" / family history / habits
    smoking_status      VARCHAR(64),
    alcohol_use         VARCHAR(64),
    exercise_frequency  VARCHAR(64),
    chronic_conditions  TEXT,        -- JSON or comma-separated
    medications         TEXT,        -- JSON or comma-separated
    allergies           TEXT,        -- JSON or comma-separated

    mental_health_notes TEXT,
    sleep_hours         NUMERIC(3,1),

    -- Reference to a summary file in Drive (optional)
    profile_doc_file_id VARCHAR(255),

    created_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP NOT NULL DEFAULT NOW()
);
```

If you want to be more structured, you can use JSON fields (e.g. `health_data JSONB`) instead of many columns.

## 6.4 Documents (generic index)

Instead of separate tables per box for files, use one `documents` table:

```sql
CREATE TYPE document_box_type AS ENUM ('profile', 'exam', 'appointment', 'prescription', 'vaccine', 'other');

CREATE TABLE documents (
    id              SERIAL PRIMARY KEY,
    user_id         INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    box_type        document_box_type NOT NULL,

    drive_file_id   VARCHAR(255) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    issue_date      DATE,          -- date on the document (exam date, prescription date, etc.)
    upload_date     TIMESTAMP NOT NULL DEFAULT NOW(),

    status          VARCHAR(32),   -- e.g. pending/scheduled/completed
    doctor_name     VARCHAR(255),
    clinic_name     VARCHAR(255),
    tags            TEXT,          -- optional comma-separated or JSON

    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## 6.5 Exams

Store exam-specific metadata and tie to a `documents` record:

```sql
CREATE TYPE exam_status AS ENUM ('pending', 'scheduled', 'completed', 'cancelled');

CREATE TABLE exams (
    id                  SERIAL PRIMARY KEY,
    user_id             INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    request_doc_id      INTEGER REFERENCES documents(id),  -- exam order
    result_doc_id       INTEGER REFERENCES documents(id),  -- result doc
    exam_type           VARCHAR(255),
    status              exam_status NOT NULL DEFAULT 'pending',

    scheduled_at        TIMESTAMP,          -- date/time of exam
    calendar_event_id   VARCHAR(255),       -- Google Calendar event ID

    notes               TEXT,

    created_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## 6.6 Appointments

```sql
CREATE TYPE appointment_status AS ENUM ('upcoming', 'completed', 'cancelled', 'no_show');

CREATE TABLE appointments (
    id                  SERIAL PRIMARY KEY,
    user_id             INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    doctor_name         VARCHAR(255),
    specialty           VARCHAR(255),
    location            VARCHAR(255),

    scheduled_at        TIMESTAMP NOT NULL,
    status              appointment_status NOT NULL DEFAULT 'upcoming',

    calendar_event_id   VARCHAR(255),
    notes               TEXT,
    related_doc_id      INTEGER REFERENCES documents(id),

    created_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## 6.7 Vaccines

```sql
CREATE TABLE vaccines (
    id                  SERIAL PRIMARY KEY,
    user_id             INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    vaccine_name        VARCHAR(255) NOT NULL,
    dose_number         INTEGER,
    date_administered   DATE,
    location            VARCHAR(255),
    notes               TEXT,

    certificate_doc_id  INTEGER REFERENCES documents(id),  -- vaccine proof

    created_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## 6.8 Consent & Settings

```sql
CREATE TABLE user_settings (
    user_id                     INTEGER PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    allow_anonymous_analytics   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at                  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

# 7. NestJS Module / API Design

## 7.1 Modules

* `AuthModule`

  * `/auth/google` → accepts ID token, returns app JWT.
  * Guard for JWT validation.

* `UsersModule`

  * `/users/me` → get profile.
  * `/users/me/settings` → update consent, timezone, etc.

* `HealthProfileModule`

  * `/profile` (GET/PUT) → manage health profile.
  * Internal service to update profile Doc in Drive if used.

* `DriveModule` (internal-only)

  * Services for:

    * Create root & folders.
    * Upload files.
    * Update file properties.

* `DocumentsModule`

  * `/documents` (GET) → list by box, filters.
  * `/documents` (POST) → upload and create doc record.
  * `/documents/:id` (GET/PUT/DELETE)`.

* `ExamsModule`

  * `/exams` (GET/POST/PATCH)` – create exam record, link request/result docs.
  * Logic to call `CalendarModule` if scheduling.

* `AppointmentsModule`

  * `/appointments` (GET/POST/PATCH)` – create/manage appointments.
  * Calls `CalendarModule` to create/update events.

* `VaccinesModule`

  * `/vaccines` (GET/POST/PATCH)` – manage vaccinations and linked certificates.

* `CalendarModule` (internal)

  * Wrapper for Calendar API calls.

---

# 8. Frontend (Angular) Considerations

* **App shell**

  * Login page with Google button.
  * Main layout with sidebar tabs: Profile, Exams, Appointments, Prescriptions, Vaccines, Settings.

* **State management**

  * Use a state management solution (NgRx or simple services) to keep user data and lists.

* **Forms**

  * Multi-step form for health profile.
  * Reusable upload component for documents.

* **Drive integration UX**

  * Show “Open in Google Drive” buttons for each doc.
  * For power users: optional “View in Google Drive folder” link.

---

# 9. Security, Privacy, and Compliance Notes

* HTTPS everywhere.
* Encrypt refresh tokens at rest.
* Limit logs: no medical content or raw tokens in logs.
* Clear privacy notice: medical data is stored in user’s Google account (Drive/Calendar), with your app holding metadata.
* Plan for Google OAuth App verification (sensitive scopes).
