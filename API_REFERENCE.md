# AediaX AI HR Backend – API Reference

Base URL (local): `http://localhost:5000/api`

## 1. Common rules

### Authentication
Most routes need a **Firebase ID token**:

```
Authorization: Bearer <FIREBASE_ID_TOKEN>
```

- Get the token on the frontend: `await firebase.auth().currentUser.getIdToken()`
- Routes marked **Verified** also need a verified email (otherwise `403 EMAIL_NOT_VERIFIED`).
- Routes marked **Public** need no token. **Optional** means the token is used only if sent.

### Roles
`SUPER_ADMIN`, `HR_ADMIN`, `HR_MANAGER`, `INTERNSHIP_COORDINATOR`, `MENTOR`, `INTERN`

In the tables below, **HR** = `SUPER_ADMIN`, `HR_ADMIN`, `HR_MANAGER`, `INTERNSHIP_COORDINATOR`.

### Response format
Success:
```json
{ "data": { }, "meta": { "requestId": "..." } }
```
Lists add pagination to `meta`: `page, pageSize, total, hasNextPage`.

Error:
```json
{ "error": { "code": "NOT_FOUND", "message": "..." }, "meta": { "requestId": "..." } }
```
Common codes: `UNAUTHENTICATED (401)`, `FORBIDDEN (403)`, `EMAIL_NOT_VERIFIED (403)`, `NOT_FOUND (404)`, `INVALID_STATE / CONFLICT (409)`, `VALIDATION_ERROR (422)`, `RATE_LIMITED (429)`.

### List query parameters
| Param | Default | Notes |
|---|---|---|
| `page` | 1 | |
| `pageSize` | 20 | max 100 |
| `q` | – | search text |
| `status` | – | filter |
| `department` | – | filter |
| `sort` | `createdAt` | |
| `order` | `desc` | `asc` / `desc` |

---

## 2. Health and service info

| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/` (server root, not `/api`) | Public | Service name, version |
| GET | `/api/health` | Public | Health check |

```bash
curl http://localhost:5000/api/health
```

---

## 3. Auth – `/api/auth`

| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/auth/me` | Token | Current user (uid, email, role, profile) |
| POST | `/auth/session` | Verified | Confirms the session, returns uid and role (201) |
| POST | `/auth/revoke` | Token | Returns 204 (placeholder, no server-side revoke yet) |

```bash
curl http://localhost:5000/api/auth/me -H "Authorization: Bearer $TOKEN"
```

---

## 4. Public internships – `/api/internships`

| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/internships` | Optional | List postings (defaults to `status=active`) |
| GET | `/internships/:id` | Optional | One posting |

```bash
curl "http://localhost:5000/api/internships?q=react&department=Engineering&page=1&pageSize=10"
```

---

## 5. Intern self-service – `/api/me` (Verified, any logged-in user)

| Method | Path | Description |
|---|---|---|
| GET | `/me/profile` | My profile |
| PUT | `/me/profile` | Update my profile |
| GET | `/me/dashboard` | Profile, internship, attendance, task progress %, documents, notifications |
| GET | `/me/applications` | My applications (paginated) |
| GET | `/me/tasks` | My tasks |
| GET | `/me/attendance` | My attendance |
| GET | `/me/progressReports` | My progress reports |
| GET | `/me/documents` | My documents |
| GET | `/me/notifications` | My notifications |
| PATCH | `/me/notifications/:id/read` | Mark a notification as read |

`PUT /me/profile` body (all fields optional, no extra fields allowed):
```json
{
  "displayName": "Rahul Kumar",
  "phone": "9876543210",
  "college": "NIT Rourkela",
  "course": "B.Tech CSE",
  "graduationYear": 2027,
  "skills": ["react", "node", "sql"]
}
```

---

## 6. Intern applications – `/api/applications` (Verified)

| Method | Path | Description |
|---|---|---|
| POST | `/applications` | Submit a new application (201). `409` if an active one already exists for that internship |
| GET | `/applications/:id` | Get my application |
| PATCH | `/applications/:id` | Edit (only in `draft` or `correction_requested`) |
| POST | `/applications/:id/submit` | Submit |
| POST | `/applications/:id/withdraw` | Withdraw |

Create body:
```json
{
  "internshipId": "abc123",
  "personalDetails": { "name": "Rahul Kumar", "phone": "9876543210" },
  "academicDetails": { "college": "NIT Rourkela", "cgpa": 8.4 },
  "preferredRole": "Frontend Intern",
  "skills": ["react", "css"],
  "resumeDocumentId": "doc123",
  "declarationAccepted": true
}
```
`declarationAccepted` must be exactly `true`.

---

## 7. Public document verification – `/api/documents`

| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/documents/verify/:identifier` | Public | Verify a certificate or ID card by its identifier |

```bash
curl http://localhost:5000/api/documents/verify/CERT-2026-0001
```

---

## 8. Admin – `/api/admin` (Verified + role check)

### Dashboard
| Method | Path | Roles | Description |
|---|---|---|---|
| GET | `/admin/dashboard` | SUPER_ADMIN, HR_ADMIN, HR_MANAGER | Counts of users, applications, postings, tasks, documents |

### Applications
| Method | Path | Roles | Description |
|---|---|---|---|
| GET | `/admin/applications` | HR | List (supports list params) |
| GET | `/admin/applications/:id` | HR | One application |
| PATCH | `/admin/applications/:id/status` | HR | Change status |
| POST | `/admin/applications/:id/request-correction` | HR | Ask the applicant to correct |
| POST | `/admin/applications/:id/assign` | HR | Assign department, manager, mentor |

Status change body, and the allowed transitions:
```json
{ "to": "approved", "reason": "Strong profile" }
```
| From | Allowed `to` |
|---|---|
| `draft` | `submitted` |
| `submitted` | `under_review`, `correction_requested`, `approved`, `rejected` |
| `under_review` | `correction_requested`, `approved`, `rejected` |
| `correction_requested` | `submitted`, `under_review` |
| `approved`, `rejected`, `withdrawn` | none (final) |

Anything else returns `409 INVALID_STATE`.

Request correction body: `{ "reason": "Upload a clearer resume" }`
Assign body: `{ "department": "Engineering", "managerId": "u1", "mentorId": "u2" }`

### Internship postings
| Method | Path | Roles | Description |
|---|---|---|---|
| GET | `/admin/internships` | HR | List |
| POST | `/admin/internships` | HR | Create (201) |
| GET | `/admin/internships/:id` | HR | One |
| PATCH | `/admin/internships/:id` | HR | Update fields |
| PATCH | `/admin/internships/:id/status` | HR | Body `{ "status": "active" }` |
| POST | `/admin/internships/:id/extend` | SUPER_ADMIN, HR_ADMIN, HR_MANAGER | Body is saved as `extensionRequest`, status becomes `extended` |
| POST | `/admin/internships/:id/complete` | SUPER_ADMIN, HR_ADMIN, HR_MANAGER | Marks `completed`, 100% |

### Tasks
Roles: SUPER_ADMIN, HR_ADMIN, HR_MANAGER, MENTOR (delete excludes MENTOR)

| Method | Path | Description |
|---|---|---|
| GET | `/admin/tasks` | List |
| POST | `/admin/tasks` | Create (default status `pending`) |
| PATCH | `/admin/tasks/:id` | Update |
| DELETE | `/admin/tasks/:id` | Delete (204) |
| POST | `/admin/tasks/:id/review` | Body `{ "status": "reviewed", "reviewNote": "Good work" }` |

### Attendance
| Method | Path | Roles | Description |
|---|---|---|---|
| GET | `/admin/attendance` | SUPER_ADMIN, HR_ADMIN, HR_MANAGER, MENTOR | List |
| POST | `/admin/attendance` | HR | Create record |
| PATCH | `/admin/attendance/:id` | HR | Update record |

### Documents (ID cards and certificates)
| Method | Path | Roles | Description |
|---|---|---|---|
| GET | `/admin/documents` | SUPER_ADMIN, HR_ADMIN, INTERNSHIP_COORDINATOR | List issued documents |
| POST | `/admin/documents/id-card/preview` | HR | Draft preview, nothing saved |
| POST | `/admin/documents/certificate/preview` | SUPER_ADMIN, HR_ADMIN, HR_MANAGER | Draft preview, nothing saved |
| POST | `/admin/documents/id-card/issue` | SUPER_ADMIN, HR_ADMIN | Issue an ID card (201) |
| POST | `/admin/documents/certificate/issue` | SUPER_ADMIN, HR_ADMIN | Issue a certificate (201) |
| POST | `/admin/documents/:id/revoke` | SUPER_ADMIN, HR_ADMIN | Body `{ "reason": "Issued by mistake" }` |

### Notifications and announcements
| Method | Path | Roles | Description |
|---|---|---|---|
| GET | `/admin/notifications` | HR | List |
| POST | `/admin/notifications` | HR | Create (201) |
| GET | `/admin/announcements` | HR | List |
| POST | `/admin/announcements` | SUPER_ADMIN, HR_ADMIN | Publish (201) |

### Logs and reports
| Method | Path | Roles | Description |
|---|---|---|---|
| GET | `/admin/logs/audit` | SUPER_ADMIN, HR_ADMIN | Audit log list |
| GET | `/admin/reports/interns.csv` | SUPER_ADMIN, HR_ADMIN, INTERNSHIP_COORDINATOR | CSV of users (id, email, displayName, role, college, course) |

```bash
curl -X PATCH http://localhost:5000/api/admin/applications/APP_ID/status \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"to":"under_review"}'

curl http://localhost:5000/api/admin/reports/interns.csv \
  -H "Authorization: Bearer $TOKEN" -o interns.csv
```

---

## 9. AI / Mini-NLP – `/api/admin/ai` (Verified, HR roles)

Runs locally and offline (no external AI calls). Rate limit: 60 requests per window.
Every response has this shape and is advisory only (`requiresHumanReview` is always `true`):

```json
{
  "data": {
    "model": "aediax-mini-nlp-v2",
    "generatedAt": "2026-10-06T10:00:00.000Z",
    "confidence": 0.82,
    "requiresHumanReview": true,
    "action": "resume-extract",
    "result": { }
  },
  "meta": { "requestId": "..." }
}
```

| Method | Path | Description |
|---|---|---|
| GET | `/admin/ai/actions` | List available AI actions |
| POST | `/admin/ai/resume-extract` | Parse resume text |
| POST | `/admin/ai/match-candidate` | Rank candidates against internships |
| POST | `/admin/ai/search` | Natural-language query to intent and filters |
| POST | `/admin/ai/summarize-application` | Summarise an application |
| POST | `/admin/ai/summarize-progress` | Summarise tasks, reports and attendance |
| POST | `/admin/ai/classify-feedback` | Classify feedback text |
| POST | `/admin/ai/recommend-next-action` | Suggest next step for an application status |

### 9.1 `resume-extract`
Body: `text` or `resumeText` (max 20000 chars, one is required).
```json
{ "text": "Rahul Kumar\nrahul@example.com\nB.Tech CSE, NIT Rourkela\nSkills: React, Node.js, SQL" }
```
Result includes contact info, name, links, sections, education, experience, projects, certifications and skills.

### 9.2 `match-candidate`
Provide one or more candidates AND one or more internships. Max 500 candidate x internship pairs. `limit` 1–200 (default 50).
```json
{
  "candidates": [
    { "id": "c1", "preferredRole": "Frontend", "skills": ["react", "css"], "resumeText": "..." }
  ],
  "internship": {
    "id": "i1", "title": "Frontend Intern",
    "requiredSkills": ["react", "javascript"], "preferredSkills": ["typescript"]
  },
  "limit": 10
}
```
Result: `{ "matches": [ ... ] }` ranked best first.

### 9.3 `search`
Body: `query` or `text` (max 500 chars).
```json
{ "query": "show react candidates" }
```
Result: `intent`, `intentConfidence`, `matchedPatterns`, `alternatives`, `collection`, `filters`, `answer`.
Intents: `candidate_search`, `application_status`, `attendance`, `task_progress`, `performance`, `document_verification`, `internship_search`, `general`.

### 9.4 `summarize-application`
Body: `application` (object or string) or `text`.
```json
{ "application": { "preferredRole": "Frontend", "skills": ["react"], "academicDetails": { "college": "NIT" } } }
```

### 9.5 `summarize-progress`
Body (all optional, default empty arrays):
```json
{
  "tasks": [{ "title": "Build login page", "status": "completed" }],
  "reports": [],
  "attendance": [{ "date": "2026-10-01", "status": "present" }]
}
```

### 9.6 `classify-feedback`
Body: `text` or `feedback` (max 5000 chars).
```json
{ "text": "Great communication, but often misses deadlines." }
```

### 9.7 `recommend-next-action`
Body: any object, usually with `status`.
```json
{ "status": "under_review" }
```

```bash
curl -X POST http://localhost:5000/api/admin/ai/search \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"query":"show react candidates"}'
```

---

## 10. Using the API from JavaScript

```js
const BASE = 'http://localhost:5000/api';

async function api(path, { method = 'GET', body } = {}) {
  const token = await firebase.auth().currentUser.getIdToken();
  const res = await fetch(BASE + path, {
    method,
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  if (res.status === 204) return null;
  const json = await res.json();
  if (!res.ok) throw new Error(json.error?.message || 'Request failed');
  return json;
}

// examples
const { data } = await api('/me/profile');
await api('/applications', { method: 'POST', body: { /* see section 6 */ } });
const ai = await api('/admin/ai/search', { method: 'POST', body: { query: 'show react candidates' } });
```

## 11. Quick total

About **64 endpoints**: 2 health, 3 auth, 2 public internships, 10 intern self-service, 5 applications, 1 public verify, 33 admin, 8 AI.
