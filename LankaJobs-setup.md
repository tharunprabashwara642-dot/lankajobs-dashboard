# LankaJobs Admin and shared services

## Components

- `admin/` is a responsive Vite/React admin site. It uses Supabase Auth and the public publishable/anon key only. Row-level security protects all database access.
- `supabase/migrations/` creates the database, roles, audit trail, policies, categories, and safe disabled-by-default monetization settings.
- `supabase/functions/payhere-create` creates a pending order and signed PayHere checkout on the server.
- `supabase/functions/payhere-notify` validates PayHere's MD5 callback signature and order amount/currency before making an idempotent status transition.
- The Android app pulls published rows from the public Supabase REST API at launch into Room. Room remains its offline cache; bundled demo jobs remain as a no-backend fallback. Drafts never appear in the public API.
- The Android Home feed has a labeled Google Mobile Ads banner. During setup it defaults to Google's test app/unit IDs; replace them with your real AdMob IDs for release builds. Official apply/company links stay on each job and are not treated as ads.

## Supabase setup

1. Create a Supabase project, then deploy the migration:

   ```powershell
   supabase login
   supabase link --project-ref YOUR_PROJECT_REF
   supabase db push
   ```

   Or paste `supabase/migrations/202609240001_initial.sql` into the Supabase SQL Editor.
2. In Supabase Authentication, create your administrator account. Signup is disabled for the project.
3. In SQL Editor, grant the first administrator role:

   ```sql
   insert into public.user_roles(user_id, role)
   values ('AUTH_USER_UUID', 'admin')
   on conflict (user_id) do update set role = 'admin';
   update public.profiles set role='admin' where id='AUTH_USER_UUID';
   ```

   Get the UUID from Authentication → Users. New Auth users get a profile and non-admin role automatically. Other admins can grant roles from the Users page.
4. For local admin development, copy `admin/.env.example` to `admin/.env.local`, set `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` (or publishable key), then run `npm install` and `npm run dev` in `admin/`. Open `http://localhost:5173/admin/`. `npm run build` creates `admin/dist/`; publish its contents under `/admin/` on a static host with SPA fallback to `index.html`. Configure the deployed admin origin in Supabase Auth redirect URLs.
5. Set the Android Gradle properties `supabaseUrl` and `supabaseAnonKey`, or environment variables `SUPABASE_URL` and `SUPABASE_ANON_KEY`, before building. These are public-client values; never use the service-role key here.

## PayHere setup

Deploy both functions with the Supabase CLI. Add Edge Function secrets on the server:

```text
PAYHERE_MERCHANT_ID=your merchant id
PAYHERE_MERCHANT_SECRET=your merchant secret
PAYHERE_MODE=sandbox
PAYHERE_NOTIFY_URL=https://YOUR_PROJECT.supabase.co/functions/v1/payhere-notify
PAYHERE_RETURN_URL=https://your-site.example/payment/return
PAYHERE_CANCEL_URL=https://your-site.example/payment/cancel
ADMIN_ORIGIN=https://your-admin-site.example
```

Then run:

```powershell
supabase functions deploy payhere-create
supabase functions deploy payhere-notify
```

Supabase supplies `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY` to deployed functions. The service-role key is read only inside the server-side callback function and must never be added to the admin `.env` or Android build. Test using PayHere sandbox credentials; switch `PAYHERE_MODE=live` only after merchant approval and callback testing. Create package prices in the admin Payment packages page; the server reads the active package amount and currency, ignoring client-supplied amounts. The create function authorizes the signed-in employer/owner before creating an order. Do not manually mark an order paid; PayHere callbacks are signature- and amount-verified.

## AdMob setup

Create the Android app and banner unit in AdMob, then supply `admobAppId` and `admobBannerUnitId` Gradle properties or `ADMOB_APP_ID` and `ADMOB_BANNER_UNIT_ID` environment variables for release builds. The checked-in defaults are Google test IDs, not production IDs. The admin's `advertising` app setting is a record for operations/configuration; AdMob serves the Android banner through its SDK. Keep ad settings separate from job apply URLs.

## Environment notes

The repository does not include project-specific Supabase, PayHere, or AdMob credentials or a deployed Supabase project. Until those are supplied and deployed, the admin shows a setup prompt, Android stays on its Room cache/demo seed, and payment settings remain disabled in sandbox mode. Create no public admin account; use Supabase Auth and assign the admin role explicitly.
