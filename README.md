# Document2Action — Frontend

> Turn confusing documents into clear actions.

React 18 + Vite + React Router. Plain JavaScript.

## Status: document chat + final integration — frontend complete, backend NOT verified

The product flow is wired end to end on the frontend: **upload → processing → analysis → action plan → tasks → document-specific chat**.
Every value shown comes from the backend; where the backend can't answer, the UI says so and nothing is invented.

No real backend exists yet, so none of it has run against one. A working product needs a backend implementing the
[Authentication](#authentication-contract), [Upload & processing](#upload--processing-contract), [Document analysis](#document-analysis-contract)
and [Document chat](#document-chat-contract) contracts. Nothing is stored in the browser.

Not built yet (planned): filters/sorting for documents and tasks, settings persistence, document preview, server-side chat history.

## Run

```bash
npm install
cp .env.example .env     # then set VITE_API_BASE_URL (optional while there is no backend)
npm run dev
npm run check            # dependency-free static checks
npm run build
```

`VITE_API_BASE_URL` is the API root including any prefix, no trailing slash (e.g. `http://localhost:8000/api`). It is public (every `VITE_*` value is embedded in the bundle) — never put secrets in it.
Production hosting must serve `index.html` for unknown paths (SPA fallback) because the app uses real URLs.

## Routes

| Path | Access | Screen |
|---|---|---|
| `/login` | signed-out only | Sign in |
| `/signup` | signed-out only | Create account |
| `/` | **private** | Home: hero, upload, progress stats, recent documents, next-up tasks |
| `/documents` | **private** | My Documents (`?q=` search) |
| `/documents/:documentId` | **private** | Document workspace shell (analysis view is Step 4) |
| `/tasks` | **private** | My Tasks |
| `/history` | **private** | History |
| `/settings` | **private** | Settings (shows the real account; editing unavailable) |
| anything else | **private** | Not found (signed-out visitors are sent to `/login` first) |

**Protection behaviour**

- While the session is being checked, a full-screen "Checking your session…" is shown — no private UI renders.
- Signed out → redirect to `/login`, remembering the requested route; after sign-in the user returns to it
  (in-app paths only; never `/login`, `/signup` or an external URL). After an explicit sign-out nothing is remembered.
- Signed in visiting `/login` or `/signup` → sent into the app.
- Session check failed (server down / malformed reply) → an error screen with **Try again** (not a redirect to `/login`).
- Session ended server-side (HTTP 401 on any private request) → `/login` with "Your session has expired."
- Private data providers (documents, tasks) are mounted only inside the guard and keyed by user id, so signing out,
  expiry, or switching user destroys all loaded data and aborts in-flight requests.

## Structure

```
src/
  constants/   routes, upload hints
  models/      data shapes + normalizers (document, task, user, analysis) (ISO dates; missing => null, never invented)
  utils/       dates (strict ISO), fileValidation, validation (forms), navigation, cx
  services/    apiClient (only reader of the base URL, only caller of fetch/XHR), typed errors,
               authService, documents/tasks/chat services
  state/       AuthContext (session), lazy shared resources (documents, tasks), ChatContext (per-document transcripts), ingestMachine (pure
               upload/processing state machine), useDocumentUpload, useDocumentTracker (bounded polling),
               useTaskToggle, usePreventFileDropNavigation
  components/
    ui/        Button Card Badge Input PasswordInput Checkbox Switch Modal Notice EmptyState LoadingState ErrorState Toast Icon
    auth/      ProtectedRoute PublicOnlyRoute AuthLayout FullPageState
    layout/    AppShell Sidebar Topbar PageHeader BackgroundDecor
    insights/  DocumentInsights InsightSection ActionPlan   (renders backend analysis only)
    …          ResourceView, DocumentRow, DocumentFacts, ProcessingTracker, TaskItem, StatCard,
               UploadDropzone, UploadButton, ChatPanel (document chat), HeroIllustration
  pages/       one file per route
  styles/      tokens (the only file with raw colours) → base → layout → ui → components → pages
scripts/static-check.mjs
```

## Authentication contract

**UNVERIFIED — this is what the frontend implements and requires of the backend.** Paths are relative to
`VITE_API_BASE_URL`. The four paths come from the original project brief; everything else below is the frontend's
documented expectation, not discovered backend behaviour.

### Session mechanism

- The session is an **HttpOnly cookie set by the backend** on login/signup and cleared on logout.
- The frontend never reads, stores or sends tokens by hand; no `localStorage`/`sessionStorage`/`document.cookie`.
- Every request uses `fetch(..., { credentials: "include" })` (centralised in `src/services/apiClient.js`).
- **Identity comes only from `GET /auth/me`.** Bodies of login/signup are ignored, so a misconfigured cookie can never
  produce a "signed in" UI.

### Endpoints

| Method & path | Request body | Success | Failure the frontend understands |
|---|---|---|---|
| `POST /auth/login` | `{ "email": string, "password": string }` (JSON) | `2xx`; sets the session cookie; body ignored | `401`/`400` → "Incorrect email or password." · `429` → "Too many attempts…" · `5xx` → generic server message · network failure |
| `POST /auth/signup` | `{ "email": string, "password": string, "name"?: string }` — `name` is sent **only if non-empty** | `2xx`; **may** set the session cookie; body ignored | `409` → "An account with this email already exists." · `400`/`422` → the response's `message` is shown · `429`, `5xx` as above |
| `GET /auth/me` | – | `200` with the user (below) | `401` = no/expired session. Anything else (5xx, network, non-user body) = "couldn't check your session" with retry |
| `POST /auth/logout` | – | `2xx`/`204`; invalidates the session and clears the cookie | `401` is treated as already signed out. Any other failure keeps the user signed in and shows an error |

**User shape** (`GET /auth/me`) — the bare object or wrapped as `{ "user": { … } }`:

```json
{ "id": "string | number", "email": "string", "name": "string | null (optional)" }
```

`id` and `email` are required; a response without them is reported as malformed ("couldn't read"). Extra fields
(roles, avatars, …) are ignored. The UI shows the name, or the email if there is no name; initials are derived from those.

**Error bodies:** JSON `{ "message": string }` (or `{ "error": string }`) is used where a server message is displayed.

### Frontend flows

| Flow | What happens |
|---|---|
| App start / reload | `GET /auth/me` → authenticated · `401` → signed out · no base URL → "unavailable" · other failure → error screen |
| Login | validate → `POST /auth/login` → **`GET /auth/me`** → authenticated. If `/auth/me` is `401` after a successful login: "We signed you in, but couldn't start your session." (never a fake success) |
| Signup | validate → `POST /auth/signup` → `GET /auth/me`: session exists → authenticated; `401` → "Account created. Please sign in." on `/login` |
| Logout | `POST /auth/logout`; only after the server confirms is the state cleared and the user sent to `/login` |
| Expiry | any non-`/auth/*` request answered `401` ends the session (`403` does not) |

### What the backend must provide (otherwise sign-in cannot work)

1. The four endpoints above, with these JSON shapes and status codes.
2. A session cookie: `HttpOnly`; `Secure` in production; a deliberate `SameSite` choice.
3. If the frontend and API are on **different origins**: CORS with an explicit `Access-Control-Allow-Origin: <frontend origin>`
   (not `*`), `Access-Control-Allow-Credentials: true`, and allowed methods `GET, POST, PATCH, DELETE, OPTIONS`
   and headers `Content-Type, Accept`. Alternatively serve the API under the same origin (e.g. a reverse proxy at `/api`),
   which avoids CORS entirely — `VITE_API_BASE_URL` may be a relative path such as `/api`.
4. CSRF protection appropriate to cookie sessions (e.g. `SameSite=Lax/Strict` and/or a token). **The frontend does not
   implement a CSRF token** because no scheme has been specified.
5. Password rules enforced server-side. The frontend only requires 8+ characters as a convenience
   (`src/constants/auth.js`); the server's message is displayed if it disagrees.

## Upload & processing contract

**UNVERIFIED — what the frontend implements and requires.** Paths are relative to `VITE_API_BASE_URL`; authentication is the
same HttpOnly session cookie as everywhere else (uploads use XHR with `withCredentials`, so the cookie is sent).
A `401` ends the session exactly like any other private request.

### Endpoints

| Method & path | Request | Success | Used for |
|---|---|---|---|
| `POST /documents` | `multipart/form-data`, **one** file in the field named `file` (field name assumed) | `201`/`200` + the created document (bare, or `{ "document": {…} }`) | Upload |
| `GET /documents/:id` | – | the document (bare or wrapped) | Opening a document **and** following processing (no separate status endpoint is assumed) |
| `POST /documents/:id/analyze` | – | `2xx` (empty body or the document) | **Optional** "Retry analysis" for a failed document. `404`/`405`/`501` → the UI says retrying isn't supported |
| `GET /documents` | – | `[doc]` or `{ "documents": [doc] }` | Lists, dashboard |

### Document shape (fields the frontend reads; anything else is ignored)

```json
{
  "id": "string | number",                      // required
  "title": "string",                            // expected: the file name / display name
  "documentType": "string | null",
  "uploadedAt": "ISO-8601 timestamp",
  "analysisStatus": "uploaded | processing | analyzing | ready | failed",
  "failureReason": "string | null",             // shown when status is failed; otherwise a generic message
  "fileSizeBytes": 123456,
  "priority": "high | medium | low | null",
  "deadline": { "date": "ISO date", "label": "string?" } | null,
  "pendingActionCount": 0
}
```

A missing field is treated as unknown — never defaulted. If `analysisStatus` is absent the page says the server hasn't reported
a status and does not poll. The created-document response must at least contain `id`; a 2xx reply without one is reported as
"the upload may have worked… check My Documents" (and is not retried automatically, to avoid duplicates).

### Error handling the frontend understands (POST /documents)

| Response | Message / behaviour | "Try again" offered |
|---|---|---|
| `400` | server `message`, else "couldn't accept this file" | no |
| `401` | session ends → `/login` ("Your session has expired") | – |
| `403` | "You don't have permission to upload documents" | no |
| `413` | "The server says this file is too large" | no |
| `415` | "The server doesn't accept this type of file" | no |
| `422` | server `message`, else "couldn't process this file" | no |
| `429` | "Too many uploads. Please wait…" | **yes** |
| `5xx` | generic "server ran into a problem" (raw server text is never shown) | **yes** |
| network failure | "couldn't reach the server" | **yes** |
| non-JSON / id-less 2xx | "upload may have worked… check My Documents" | no |

### Client-side validation (before anything is sent)

Applies to the file picker **and** drag & drop (the `accept` attribute is only a hint):

- one file at a time; types **PDF, JPG/JPEG, PNG, TXT**; the file's MIME type and extension must agree (an empty MIME type falls back to the extension);
- not empty; **≤ 10 MB** (binary megabytes: 10 × 1024 × 1024 bytes);
- the leading bytes must match the claimed type (`%PDF-`, JPEG `FF D8 FF`, PNG signature; text must contain no NUL bytes).

The server must validate as well; client checks only prevent pointless uploads.

### State machine (`src/state/ingestMachine.js`, pure and unit-tested)

```
idle ─SELECT→ validating ─UPLOAD_STARTED→ uploading ─UPLOAD_SUCCEEDED→ processing | analyzing | ready | error | unknown
                 │INVALID                     │UPLOAD_FAILED
                 ▼                             ▼
               error ◀────────────────────────┘                  CANCEL → idle · RESET → idle
any non-busy phase ─DOCUMENT_STATUS→ processing | analyzing | ready | error | unknown   (server-reported status)
```

`uploaded`/`processing` → **processing**, `analyzing` → **analyzing**, `ready` → **ready**, `failed` → **error** (kind `processing`),
no status → **unknown**. Selections while `validating`/`uploading` are ignored (no double submission) and stale events are dropped.

### Flow & polling

1. Upload shows real progress (`xhr.upload.onprogress`, only when the browser can compute a length) and a **Cancel** button.
2. On success the server's document is added to the shared list (then quietly re-fetched) and the user is taken to `/documents/:id`.
3. That page follows `GET /documents/:id`: first poll after 2 s, then 3 s, 5 s, 8 s (repeating). Polling **stops** on ready / failed / unknown,
   on leaving the page, after **5 minutes** ("taking longer than expected" + *Check again*), after **3 consecutive failures**
   ("lost contact" + *Check again*), or when the document is gone (404). It **pauses while the tab is hidden**. One request is in flight at a time.
4. Navigating away mid-upload aborts the upload (the server may or may not have received it — check My Documents).

### What the backend must provide

1. `POST /documents` and `GET /documents/:id` as above, scoped to the signed-in user, `401` without a session.
2. A status that **actually changes** as the server works (`uploaded → processing → analyzing → ready | failed`). The frontend never advances it.
3. CORS (if cross-origin) allowing credentials for `POST`/`GET` and exposing nothing special; `413`/`415` for oversize/unsupported files.

## Document analysis contract

**UNVERIFIED — what the frontend implements and requires.** The analysis travels inside `GET /documents/:id` as an `analysis` object
(no separate endpoint is assumed). It is shown **only when the document's `analysisStatus` is `ready`**; while `uploaded`/`processing`/`analyzing`
or `failed`, any `analysis` the server attaches is ignored.

```json
{
  "id": "…", "title": "…", "analysisStatus": "ready",
  "analysis": {
    "summary": "string",
    "actionRequired": "string",
    "priority": "high | medium | low",
    "deadline": { "date": "2026-09-30", "label": "optional text as written in the document" },
    "requiredDocuments": [ "string" | { "name": "string", "description": "string?" } ],
    "submissionLocation": { "name": "string?", "details": "string?" },
    "warnings": [ "string" | { "message": "string", "title": "string?" } ],
    "actionPlan": [ { "id": "string|number", "title": "string", "description": "string?",
                      "priority": "high|medium|low?", "dueDate": "ISO date?", "status": "pending | completed" } ]
  }
}
```

### "Nothing" versus "unknown" — the part that must be exact

Every section is shown in one of three ways, so the UI never claims something the backend did not say:

| Section | Backend sends | UI shows |
|---|---|---|
| `deadline` | `{ "date": "<valid ISO>" }` | the date (from the ISO value), relative urgency ("Due in 3 days", "Overdue by 2 days", "Due today"), and a note when it is ≤ 7 days away or has passed |
| | `{ "label": "…" }` only (no usable date) | the label as written; **no** urgency is computed |
| | **`null`** | **"No deadline mentioned in this document."** |
| | key absent, or unreadable (e.g. `"Sep 30"`) | "Deadline information isn't available for this document." — *not* the "no deadline" claim |
| `requiredDocuments`, `warnings`, `actionPlan` | non-empty array | the items |
| | **`[]`** | "No required documents / warnings mentioned…", "No action items were provided…" |
| | absent, `null` or not an array | "…information isn't available for this document." |
| `submissionLocation` | object with `name` and/or `details` | name + details (how/where to submit belongs in `details`) |
| | **`null`** | "No submission instructions mentioned in this document." |
| | absent / empty object | "Submission information isn't available…" |
| `summary`, `actionRequired` | non-blank string | the text (line breaks kept; always rendered as plain text, never HTML) |
| | absent / blank | "A summary / The required action isn't available…" |
| `priority` | valid value | badge on *Action required*; falls back to the document's own `priority`; no badge if neither exists |

- A `ready` document **without** an `analysis` object shows "Analysis results aren't available" — no sections, nothing invented.
- A malformed `analysis` (not an object, a required document without a name, a warning without a message, an action item with an unknown `status` or no `id`)
  is reported as "the server sent a response we couldn't read" with *Try again*; items are never silently dropped.
- Text from the backend is rendered as text only (React escaping): HTML such as `<img onerror=…>` appears literally and never executes.
- Dates must be strict ISO-8601; the frontend never parses free-text dates and never searches the summary for a deadline.

### Action plan, progress and task relationships

- Action-plan item `id`s **are task ids**: ticking an item calls the existing `PATCH /tasks/:id { status }`. The item changes **only after the server confirms**
  (the checkbox is disabled while saving; a failure shows an error toast and leaves it unchanged).
- Progress ("2 of 4 actions completed", progress bar, "All actions completed.") is a plain count of the backend's own item statuses.
- Items keep the server's order; the document page never reorders them.
- The shared task list is updated immediately, and the documents list is quietly re-fetched so "N actions pending" stays correct.
  The server must therefore return the same completion state from `GET /tasks`, `GET /documents/:id` (plan) and `GET /documents` (`pendingActionCount`).
- Each task on **My Tasks** links to its source document (`documentId`, `documentTitle`); the plan on the document page is that document's tasks.

### Backend requirements added by Step 4

1. `GET /documents/:id` returns `analysis` (above) once `analysisStatus` is `ready`.
2. `PATCH /tasks/:id` accepts plan-item ids and returns the updated task (or an empty body).
3. Completion state is consistent across `/tasks`, the document's `actionPlan` and `pendingActionCount`.

## Document chat contract

**UNVERIFIED — what the frontend implements and requires.** `POST /documents/:id/chat` is the only chat endpoint the project documents; **no history
endpoint is assumed**. The document id in the path is the context: the server answers about that document only.

| Method & path | Request | Success |
|---|---|---|
| `POST /documents/:id/chat` | JSON `{ "message": string }` — exactly one field, the trimmed question | `2xx` with the assistant's reply, bare or wrapped: `{ "id"?: string\|number, "role"?: "assistant", "content": string, "createdAt"?: ISO-8601 }` or `{ "message": { … } }` |

- `content` is required and must be non-blank text. A reply that is empty, not an object, not JSON (e.g. an HTML error page) or has any other `role` is reported as
  "The assistant's reply couldn't be read" with *Try again* — it is never displayed as an answer.
- Replies are rendered as **plain text** (line breaks kept). HTML, scripts or markup in a reply — or in the user's own question — appear literally and never execute.
- Conversation memory/context beyond the single `message` is the **server's** responsibility; the frontend sends no history.

### Errors the frontend understands

| Response | Shown on the question | Retry offered |
|---|---|---|
| network failure | "couldn't reach the server" | yes |
| `400` / `422` | the server's `message`, else "couldn't process that question" | no |
| `401` | session ends → `/login` ("session has expired") | – |
| `403` | "You don't have access to chat about this document" | no |
| `404` | "This document no longer exists" | no |
| `413` | "That question is too long" | no |
| `429` | "Too many questions…" | yes |
| `5xx` | "The assistant is unavailable right now" (raw server text is never shown) | yes |
| malformed / unreadable reply | "reply couldn't be read" | yes |
| no answer within **60 s** | "taking too long to answer" (the request is aborted) | yes |

### Behaviour

- **Availability** follows only the status the server reported: `uploaded`/`processing`/`analyzing` → chat disabled ("becomes available once this document has finished
  processing"); `failed` → disabled ("couldn't be analyzed"); `ready` or **no reported status** → enabled. It becomes usable live when the server reports `ready`.
- **Scope:** the panel title area shows "About: <document title>". Each document has its own transcript; opening another document starts from *its* transcript.
  An answer that arrives after you moved to another document is stored with the **original** document only.
- **Transcript = memory only.** Kept for the session (surviving closing the panel and moving between documents); **destroyed on sign-out, session expiry, user switch and page refresh**;
  never written to browser storage. (If the server keeps history, that is not displayed — no endpoint for it is assumed.)
- **One question at a time per document** (a synchronous lock — double clicks/Enter cannot send twice). A failed question stays on screen with its error and **Try again**,
  which re-sends the *same* question once; the answer is placed directly under it. A new question can be asked after a failure.
- Questions are limited to **2000 characters** (frontend convention; the server's limit applies too). Starter prompts ("What do I need to do?", …) are *questions the user can send* — not answers.
- Accessibility: modal dialog (focus trapped, Escape closes, focus returns to "Ask about this document"), the question box takes focus on open, the transcript is a polite `role="log"`
  with "You said / Assistant said" prefixes, Enter sends and Shift+Enter inserts a line break.

### Backend requirements added by chat

1. `POST /documents/:id/chat` as above, scoped to the signed-in user's document, `401` without a session, `404` for unknown documents.
2. The reply must be the assistant's text only (no markup is interpreted).

## Data freshness (shared lists)

Documents and tasks are loaded once and shared. Screens that list them (**Home, My Documents, My Tasks, History**) **revalidate quietly when opened**:
the data already held appears instantly and is re-fetched in the background; a failed background refresh keeps the data on screen (no error replaces it), and a `401`
ends the session as usual. When a document reaches **ready with analysis** the tasks and documents lists are refreshed too — that is when the server creates the document's tasks,
so the sidebar's pending count and My Tasks are correct without a manual reload. Completing an action also quietly refreshes the documents list ("N actions pending").

## Data API the frontend expects (UNVERIFIED — adapt in `src/models/*` and `src/services/*` only)

| Call | Used by |
|---|---|
| `GET /documents` → `[doc]` or `{documents:[doc]}` | Home, Documents, History |
| `GET /documents/:id` → `doc` (+ `analysis` when ready) | Document page, processing status, insights |
| `POST /documents` (multipart, field `file`) → `doc` | Upload |
| `POST /documents/:id/analyze` → 2xx | Optional retry of a failed document |
| `POST /documents/:id/chat` `{message}` → assistant reply | Document chat |
| `GET /tasks` → `[task]` or `{tasks:[task]}` | Home, Tasks, sidebar badge |
| `PATCH /tasks/:id` `{status}` → `task` or empty | Task checkbox |

These must be scoped to the signed-in user by the backend and answer `401` when there is no session.
Field shapes are documented at the top of `src/models/document.js` and `src/models/task.js`.
Dates must be strict ISO-8601 (`2026-09-30` or a full timestamp); anything else is treated as "no date".
