# ClassPulse Attendance Web App

A production-quality attendance web application for one teacher and up to 2 classes, backed by Supabase (PostgreSQL, Row-Level Security, Realtime, Edge Functions).

---

## 1. Supabase Auth Configuration (Dashboard Settings)

Configure these settings in your Supabase project under **Authentication > Configuration**:

1. **User Signups**:
   - Turn **OFF** "Enable Email Signups" (Public self-registration must be disabled).
   - Turn **ON** "Confirm email" (or automatic confirmation handled via admin APIs).
2. **Password Security**:
   - Minimum password length: **10** characters.
   - Turn **ON** "Enable Leaked Password Protection" (Pawned passwords check).
3. **Session & Tokens**:
   - JWT Expiry Limit: **3600** seconds (1 hour).
   - Refresh Token Rotation: **ON**.
   - Refresh Token Reuse Interval: **10** seconds.

---

## 2. Teacher Account Bootstrapping

Follow these steps once to create the single teacher account:

1. In Supabase Dashboard, navigate to **Authentication > Users** and click **Add User** -> **Create User**.
2. Enter the teacher's email and a strong password (minimum 10 characters). Copy the generated user's **UUID**.
3. Open the Supabase **SQL Editor** and run the following bootstrap insert, substituting `<TEACHER_AUTH_USER_UUID>` with the UUID copied in step 2:

```sql
insert into public.profiles (id, role, full_name, must_change_password)
values (
  '<TEACHER_AUTH_USER_UUID>'::uuid,
  'teacher',
  'Prof. Alex Morgan',
  false
);
```

Students are **never** registered through public signup or manually in SQL; they are provisioned exclusively through the `provision-students` Edge Function from the Teacher dashboard.

---

## 3. Environment Variables

### Client Application (`.env` / Vercel Environment Variables)
Only these two variables are bundled into the client build:
```env
VITE_SUPABASE_URL="https://your-project-ref.supabase.co"
VITE_SUPABASE_ANON_KEY="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Supabase Edge Functions Secrets
Set these secrets in Supabase via CLI or Dashboard (**Settings > Edge Functions**). **Never** expose these in repository files or client bundles:
```bash
supabase secrets set SUPABASE_SERVICE_ROLE_KEY="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
supabase secrets set ALLOWED_ORIGIN="https://your-deployment.vercel.app"
```

---

## 4. Database Setup & Migration

Apply the single migration containing all enums, tables, check constraints, security definer triggers, view, RLS policies, and RPCs:

```bash
# Using Supabase CLI:
supabase db push

# Or run the SQL script directly in Supabase SQL Editor:
# File: supabase/migrations/20260928000000_attendance_schema.sql
```

### Scheduled Automatic Closure of Expired Sessions
If `pg_cron` extension is enabled in your database, schedule the automated closure job:
```sql
select cron.schedule('close-expired-sessions-every-minute', '* * * * *', 'select public.close_expired_sessions()');
```
*Note: Regardless of pg_cron, the Teacher UI detects expired-but-open sessions and displays an immediate "Expired: Close to finalize" button.*

---

## 5. Deploying Supabase Edge Functions

Deploy the two secure edge functions:

```bash
supabase functions deploy provision-students --no-verify-jwt=false
supabase functions deploy reset-student-password --no-verify-jwt=false
```

---

## 6. Running RLS & Acceptance Tests

Verify that all 23 security, RLS, authorization, and attendance rules pass:

```bash
# Run tests against your Supabase database:
psql "$DATABASE_URL" -f supabase/tests/rls_tests.sql
```

---

## 7. Frontend Deployment to Vercel

The application includes `vercel.json` preconfigured with single-page app rewrites and strict Content-Security-Policy (CSP) and security headers:

```bash
# Install dependencies
npm install

# Build production bundle
npm run build

# Deploy with Vercel CLI
vercel --prod
```
Ensure `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` are configured in your Vercel project settings.
