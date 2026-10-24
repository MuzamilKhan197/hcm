# HCM Lite

A MERN HR workspace for a hackathon: company setup, role-based authentication, employee records, attendance, leave, payroll, reports, notifications, and an assistant with optional backend AI integration.

## Run the demo

Requires Node.js 22.12+ and npm. From this project folder:

```sh
npm install
npm run dev:demo
```

Open **http://localhost:5173**. Choose Admin, HR, or Employee and click **Explore demo workspace**. No sample password is required. Sample users use reserved `.example` email addresses.

Demo is an explicit, development-only mode. Its data is saved to ignored `backend/data/demo.json`, not MongoDB. Restarting preserves changes but signs everyone out. The demo backend binds only to the local machine. Demo login is disabled in real MongoDB mode and production. Stop the app and remove that JSON file if you want to reset the sample workspace.

## Connect real MongoDB

1. Use a MongoDB Atlas database or a **local replica set**. Transactions are required for atomic first-run setup, employee registration, leave updates, payroll, and associated audit/notification writes. A standalone MongoDB server will fail startup with a clear message.
2. Copy `backend/.env.example` to `backend/.env` and configure:

```dotenv
PORT=4000
MONGODB_URI=mongodb://127.0.0.1:27017/hcm_lite?replicaSet=rs0
JWT_SECRET=your-random-secret-at-least-32-characters
CLIENT_URL=http://localhost:5173
NODE_ENV=development
```

Generate a strong secret locally with `node -e "console.log(require('node:crypto').randomBytes(48).toString('hex'))"`. Never commit your `.env` or credentials.

3. Run `npm run dev` and open **http://localhost:5173**.
4. Create the company and first administrator. Public setup closes after this succeeds. There are no seeded users or default passwords in MongoDB mode.
5. Create departments, then employees. Employee records support contact and emergency-contact details. Each employee receives a sign-in account with the temporary password entered during registration. Share it privately. Admin can create HR accounts; HR can create and update employee profiles but cannot change compensation or deactivate accounts. Admins can deactivate an account while retaining records, or use **Fire & permanently delete** to irreversibly remove the account, employee profile, attendance, leave, payroll/payslips, notifications, assistant/audit history, and announcements authored by that person. Salary configuration belongs to Admin.

This is a single-company application. First-run setup should be completed privately before exposing a new instance to other users. The initial administrator has no employee profile; attendance and leave are available to accounts created through employee registration.

## Commands

| Command | Purpose |
| --- | --- |
| `npm run dev:demo` | Local persistent demo API + React development server |
| `npm run dev` | Real MongoDB API + React development server |
| `npm run build` | Build the React frontend |
| `npm test` | Backend integration and payroll calculation tests |
| `npm start` | Run the local Express server |

## Separate production deployments

The existing `frontend/` and `backend/` directories are independently deployable projects. Each has its own package manifest, ignore rules, environment example, README, and Vercel configuration. The backend's root `app.js` exports Express for Vercel Functions; `server.js` remains the local development entry point. The frontend uses `VITE_API_URL` to reach the separate backend.

### Create the GitHub repositories

Create two empty repositories in your authorized GitHub account named `hcm-lite-frontend` and `hcm-lite-backend`. This workspace has no Git history to preserve. From the project root, initialize and push each directory independently (replace `<GITHUB_USERNAME>`):

```powershell
Set-Location .\frontend
git init -b main
git add .
git commit -m "Prepare HCM Lite frontend"
git remote add origin https://github.com/<GITHUB_USERNAME>/hcm-lite-frontend.git
git push -u origin main

Set-Location ..\backend
git init -b main
git add .
git commit -m "Prepare HCM Lite backend"
git remote add origin https://github.com/<GITHUB_USERNAME>/hcm-lite-backend.git
git push -u origin main
```

Never copy a local `.env` file or credentials into either repository. The committed `.env.example` files contain placeholders only.

### Deploy the backend

1. Import `hcm-lite-backend` in Vercel as the `hcm-lite-api` project with the repository root as its project root. Vercel detects `app.js` as the Express app. Use Node.js 22 or later.
2. In Vercel's Production environment, set `MONGODB_URI` to the MongoDB Atlas database URI, `JWT_SECRET` to a unique random secret of at least 32 characters, `CLIENT_URL` to the frontend's exact HTTPS origin (no path), and `NODE_ENV=production`. Optionally set `OPENAI_API_KEY` and `OPENAI_MODEL` for the assistant. Keep AI and database secrets only in the backend project's environment variables.
3. Create a least-privilege Atlas database user and configure Atlas network access for the deployment. Do not use a permissive public allowlist unless required by your Atlas plan and risk assessment.
4. Deploy and verify `https://<backend-domain>/api/health` returns `{"ok":true,"storage":"mongodb"}`.

### Deploy the frontend

1. Import `hcm-lite-frontend` as the `hcm-lite-web` Vercel project with the repository root as its project root. Set the Vite build command to `npm run build` and output directory to `dist` if Vercel does not detect them automatically.
2. Set `VITE_API_URL=https://<backend-domain>/api` in the frontend project's Production environment. This is a public URL, not a secret.
3. Deploy, open the frontend, create the company and first administrator, and verify authentication and role-restricted workflows. Set the backend's `CLIENT_URL` to the final frontend origin and redeploy the backend if that value changes.
4. Production cookies use `SameSite=None; Secure` for separate Vercel origins. Browser privacy settings can still block third-party cookies on default `*.vercel.app` domains. For reliable sign-in, use frontend and backend custom domains under the same registrable domain (for example, `app.example.com` and `api.example.com`). Add the frontend origin to `CLIENT_URL` exactly.

Preview deployments require matching Preview environment values for `VITE_API_URL` and the backend `CLIENT_URL`. Do not point public previews at production data unless intended.

## What works

- **Setup and login:** first-run company contact/timezone fields, bcrypt password hashing, JWT in HttpOnly / SameSite cookies, server-side role checks, profile editing, validated password changes that invalidate active sessions, origin checks, rate limiting, and rechecking account status on each request. New passwords must be 8–72 characters.
- **Overview:** charts and counts calculated from permitted workspace records; no hard-coded KPI increases.
- **People:** searchable directory, department/status filters, account registration, salary changes, admin-only deactivation/reactivation, and an explicitly confirmed permanent-delete action. HR does not receive salary fields.
- **Attendance:** one check-in per company date, one check-out per open record, daily views, employee history, and CSV export. A 9:15 AM threshold marks late arrivals.
- **Time off:** Annual, Sick, Casual, and Unpaid quotas; working-day calculations exclude weekends and configured holidays. Pending leave reserves balance; overlaps, past dates, and self-approval are rejected. Separate requests are required across calendar years.
- **Payroll:** monthly base salary plus allowance, joining-date proration, approved unpaid-leave deductions, an entered withholding percentage, immutable payslip snapshots, payment-status recording, PDF download, and CSV export. Duplicate employee/month payslips are prevented.
- **Company tools:** departments, announcements, future holidays, private notifications with a history page and per-user read state, announcement and payroll-generation alerts, audit log, company name editing, and role-scoped reports.
- **Assistant:** read-only answers from permitted workspace records, recorded in `aiLogs`. Without `OPENAI_API_KEY`, it uses local insights. When configured, it sends only a minimal role-scoped summary to OpenAI; it does not send names, emails, salaries, or raw records. Provider errors are surfaced rather than silently replaced with local answers.

## Access matrix

| Area | Admin | HR | Employee |
| --- | --- | --- | --- |
| Company settings / department creation | Manage | Read departments | Read departments |
| Employee directory | Manage | Register/read, no salary | Own record |
| Attendance | View all + own actions if linked | View all + own actions | Own only |
| Leave | Review others + own requests if linked | Review others + own requests | Own requests |
| Payroll | Generate / mark paid / download | No access | Own payslips |
| Announcements | Publish/read | Publish/read | Read |
| Audit log | Read | No access | No access |
| Notifications | Own | Own | Own |
| Assistant | Role-scoped summary | Role-scoped summary | Own summary |

## Structure

```text
frontend/src/
  components/    Reusable controls, dialogs, assistant
  context/       Authentication and workspace state
  hooks/         Action loading/error handling
  layouts/       Navigation and app frame
  pages/admin/   Overview, departments, payroll, settings
  pages/hr/      Employee registration and leave review
  pages/employee/ Attendance
  services/      API client and exports
  App.jsx
  main.jsx
backend/
  src/config/      MongoDB and explicit demo repositories
  src/models/      All 14 requested MongoDB collections
  src/controllers/ Authentication and setup
  src/routes/      Role-protected HR APIs
  src/middleware/  Authentication and authorization
  src/services/    Demo seed and leave defaults
  src/utils/       Dates and payroll calculations
  src/app.js
  app.js           Vercel Express export
  server.js
  test/
```

Collections: `users`, `companies`, `departments`, `designations`, `employees`, `attendance`, `leaves`, `leaveTypes`, `holidays`, `payrolls`, `notifications`, `announcements`, `auditLogs`, and `aiLogs`.

## Important MVP boundaries

Currency is PKR; timezone defaults to Asia/Karachi and can be configured using an IANA timezone. Payroll is a simple monthly calculation, **not statutory tax or labor-law compliance**. Marking a payslip paid does not transfer money. Review leave decisions before payroll generation; later changes do not rewrite snapshots. Attendance is not an automatic salary deduction. This is a single-company deployment, not multi-tenant. There is no email-based password reset, MFA, or bank integration. Registration passwords must be distributed privately. Password changes are available to signed-in users; reset delivery requires an email provider and has not been configured. The optional AI integration requires an OpenAI API key configured on the backend.

Integration tests use the same route layer with the transactional demo repository. A live MongoDB replica set must be supplied to verify its connection and transactions in your deployment. Demo mode is not a production storage option. The current workspace fetch is designed for small hackathon datasets; add pagination and bounded reporting for larger teams. In multi-instance deployment, rate-limit storage must also be shared.

Official references: [Vite setup](https://vite.dev/guide/), [Express security](https://expressjs.com/advanced/best-practice-security/), [Mongoose transactions](https://mongoosejs.com/docs/transactions.html).
