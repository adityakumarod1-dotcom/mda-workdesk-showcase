# MDA WorkDesk

Designed and developed a full-stack Operations Management Platform for Maa Durga Associates, a Petroleum and Industrial construction company, to streamline site operations across four States. The system centralizes field Employee Management, Daily Task Tracking with pending reminders , and document workflows — replacing manual spreadsheet and WhatsApp-based processes with a structured Admin and Employee Portal.

> **Note:** This is a portfolio showcase for a client project. The production codebase, business data, and client assets remain private; this repo documents the architecture, feature set, and technical decisions without exposing proprietary code or client information. All screenshots below use synthetic demo data ("Demo Employee", "Demo Fuel Station") in an isolated database — no real employee, client, or site information is shown.

**Live public site:** [associatesmaadurga.in](https://associatesmaadurga.in) — the landing page is open to everyone; the Admin/Employee portal behind it requires actual Admin/Employee credentials

---

## The flow, end to end

### 1. Public landing page
Anyone visiting `associatesmaadurga.in` sees the company's public marketing site — services, project photos, clients, state-by-state coverage, and a company timeline. This is what gets linked from LinkedIn, business cards, and search.

![Public landing page](images/01-landing-page.png)

### 2. Login
The platform uses Employee ID–based authentication instead of email login, a deliberate design decision to accommodate field staff who may not have an email address. Based on assigned role, users are directed to one of two purpose-built interfaces — an Admin dashboard for oversight and management, and an Employee portal focused on day-to-day task execution — ensuring each user only sees what's relevant to their role.

![Login screen](images/02-login.png)

New employees are issued a temporary password by an Admin and are forced through a change-password screen on first login before they can reach their dashboard.

---

### 3. Admin portal — The office side

**Dashboard** — at-a-glance counts: pending requests, active sites, employee headcount, completed sites.

![Admin dashboard](images/03-admin-dashboard.png)

**Employees** — add, edit, and deactivate staff.

![Employees list](images/04-employees-list.png)
![Add Employee form](images/05-add-employee.png)

Creating an employee auto-generates their Employee ID and a temporary password, shown once in a "save these credentials" modal — mirroring the real onboarding conversation, where a supervisor creates the account and hands the credentials to the new hire.

![Employee credentials modal](images/06-employee-credentials-modal.png)

**Sites** — register a Petrol pump, CNG station, Consumer Pump or industrial project. Each site tracks a status lifecycle (Pending Activation → Active → On Hold → Completed) and gets a dedicated Google Drive folder structure created automatically, so all site photos and documents live in Drive rather than on the app server.

![Sites list with progress bars](images/07-sites-list.png)
![Add Site form](images/08-add-site.png)

Opening a site shows its full profile, assigned employees, and live progress:

![Site Details page](images/09-site-details.png)

...alongside a complete timeline of everything that's happened on that site — registration, daily progress reports, and every checklist submission — plus the activation checklist itself (Handover, Medical Clearance, Safety Clearance, Gate Pass, Permit):

![Site timeline and daily progress](images/10-site-timeline.png)
![Activation checklist](images/11-activation-checklist.png)

**Tasks** — assign work items to employees independent of any specific site (e.g. "service the compressor at three locations this week"). Each task tracks priority, due date, and a per-assignee status through Pending → In Progress → Submitted → Approved/Rejected. Also reminders are also given at regular interval if the task are not completed and closed in the portal.

![Tasks board](images/12-tasks-board.png)
![Assign Task form](images/13-assign-task.png)

**Requests / Approvals / Inbox** — when a field employee marks a site's activation or completion checklist as done, it becomes a *request* an admin has to approve — the paper-trail control point before a site officially changes status.

**Vendors** — manages the "Our Clients & Industry Partners"  shown on the public landing page (BPCL, IOCL, Hindalco, etc. in production). Editing this list here updates the public site automatically.

![Vendors page](images/14-vendors.png)

**Reports** — generates Excel exports per site (progress, checklist status, timeline) for offline sharing with clients or internal review.

**Commercial Documents** — a general-purpose document store (contracts, GST filings, commercial paperwork), also backed by Google Drive.

**Settings** — designations, account security, and system-level configuration.

![Admin settings](images/15-admin-settings.png)

---

### 4. Employee portal — the field side

Deliberately built mobile-first (bottom tab navigation: Home / Sites / Tasks / Docs / Alerts / Settings) since this is used on phones at job sites, not desks.

![Employee dashboard](images/16-employee-dashboard.png)

**Tasks** — assigned work items with a "Start Task" action. Starting a task timestamps it and moves it to In Progress; completing one requires remarks and photo evidence, which becomes a submission for an admin reviews.

![Employee Tasks tab](images/17-employee-tasks.png)
![Task in progress, ready for completion](images/18-employee-task-in-progress.png)

**Sites** — the sites this employee is assigned to, with an "Open Site" action to view the checklist, log daily progress, and drive the site through activation and completion.

![Employee Sites tab](images/19-employee-sites.png)

Opening a site shows the same activation checklist an admin sees (read-only once submitted) and the daily progress log:

![Site activation details, employee view](images/20-employee-site-activation-view.png)
![Daily work progress](images/21-employee-daily-progress.png)

Closing out a site walks the employee through a guided completion process — upload completion photos, upload a completion certificate, add remarks — with the submit button staying disabled and a live checklist of what's still missing until everything's in place:

![Completion process — uploads](images/22-completion-process-uploads.png)
![Completion remarks and submit](images/23-completion-remarks-submit.png)

**Alerts** — push notifications (Web Push/VAPID) for new task assignments and reminders on unfinished work, so nothing depends on someone remembering to check the app.

![Employee alerts](images/24-employee-alerts.png)

**Settings** — profile info and account security, same pattern as the admin side.

![Employee profile](images/25-employee-profile.png)

---

## Architecture

```
┌─────────────────────┐        ┌──────────────────────────┐
│   React 19 Frontend  │  REST  │   Express 5 + TypeScript  │
│   Vite · Tailwind 4  │◄──────►│   Backend API              │
│   React Router 7     │        │                            │
└─────────────────────┘        └──────────┬─────────────────┘
                                            │
                        ┌───────────────────┼───────────────────┐
                        ▼                   ▼                   ▼
                 ┌─────────────┐   ┌────────────────┐   ┌───────────────┐
                 │  MongoDB    │   │  Google Drive/  │   │  Web Push      │
                 │  (Mongoose) │   │  Sheets API      │   │  (VAPID)       │
                 └─────────────┘   └────────────────┘   └───────────────┘
```

- **Monorepo** with independent `frontend/` and `backend/` workspaces
- **Auth**: JWT-based, bcrypt password hashing, role-based route guards (Admin / Employee), forced password rotation on first login
- **File storage**: uploads stream to Google Drive via a service account — no files stored on the app server, so the deployment stays stateless
- **Validation**: Zod schemas on every API boundary, plus startup-time environment-variable validation that refuses to boot on a missing or placeholder secret
- **Security**: Helmet, CORS allowlisting, rate limiting, CSP headers
- **Reporting**: server-side Excel generation via ExcelJS
- **Logging**: structured logging with Pino

## Tech stack

| Layer | Technologies |
|---|---|
| Frontend | React 19, Vite, Tailwind CSS 4, React Router 7, React Hook Form, Zod, Framer Motion, shadcn/ui |
| Backend | Node.js, Express 5, TypeScript, MongoDB + Mongoose, JWT, bcrypt |
| Integrations | Google Drive API, Google Sheets API, Web Push (VAPID), Google Maps Platform |
| Infra | Railway (deployment) |

## Notable engineering decisions

- **Fail-fast config**: the backend validates every environment variable at boot with Zod, including placeholder detection (e.g. rejects a `JWT_SECRET` that still reads `changeme`).
- **No long-lived secrets on the server filesystem in production**: the Google service account credential is injected as a raw JSON env var at deploy time rather than shipped as a file.
- **Uploads never touch the app server's disk** — files stream straight through to Google Drive, keeping the deployment stateless and easy to scale horizontally.
- **Atomic, self-healing site codes** — site codes (`ST00001`, `ST00002`, ...) are generated from a MongoDB counter document that also cross-checks against the highest existing code on every run, so the counter can't drift behind real data even after a manual DB edit.

## Live demo

Public landing page: **[associatesmaadurga.in](https://associatesmaadurga.in)** — the admin/employee portal requires real staff credentials, so this link only ever shows what's meant to be public.

---

Built by [Aditya Kumar](https://www.linkedin.com/in/aditya-kumar-a920293b9) — feel free to reach out with questions about the implementation.
