# Advance Law of Attraction – Website + Admin Panel

Your landing page with a built-in content management system. Everything on the page (text, prices,
images, skills, testimonials, bonuses, FAQs, countdown, footer, SEO) is edited at **`/admin`** and
published with one click.

## Architecture

```
Visitor ──► /                 Express renders the landing page from PUBLISHED content (PostgreSQL)
Admin   ──► /admin            Sign in → edit → Save draft → Preview → Publish
Form    ──► /api/enquiries    Leads validated and stored in PostgreSQL (+ optional email)
Images  ──► Supabase Storage  Public bucket; uploads/deletes only via the server's secret key
```

| Part | Technology |
|---|---|
| Server | Node.js 22+ (24 recommended), Express |
| Database | **Supabase PostgreSQL** (`DATABASE_URL`) |
| Images | **Supabase Storage** bucket `aloa-media` (created automatically). Local `uploads/` folder only when Supabase is not configured (development) |
| Public page | `views/landing.ejs` (your original HTML) + `public/style.css` + `public/script.js` |
| Admin panel | `admin/` – plain JavaScript, no build step |

**Security:** scrypt password hashing; random session tokens in `HttpOnly`/`Secure` cookies (only hashes stored);
CSRF token on every change; rate-limited sign-in, reset and enquiry form; uploads checked by content and scripted
SVGs refused. All Supabase tables have **row-level security enabled and no access for the `anon`/`authenticated`
roles**, so Supabase's public API cannot read admins or leads – only this server (connecting with the database
password) can. No secret is ever sent to the browser.

---

## 1. Set up Supabase (once)

1. Create a project at [supabase.com](https://supabase.com) (choose the region closest to your visitors, e.g. Mumbai).
   Save the **database password** you set.
2. **Database connection string** – click **Connect** (top of the dashboard) → **Session pooler** → copy the URI:
   ```
   postgresql://postgres.<project-ref>:<YOUR-PASSWORD>@aws-0-<region>.pooler.supabase.com:5432/postgres
   ```
   Replace `<YOUR-PASSWORD>`. If the password contains `@ : / ? # %` etc., URL-encode it (e.g. `@` → `%40`), or
   reset it to letters + numbers only.
   *Use the Session pooler – the "Direct connection" is IPv6-only and may not work from Hostinger.*
3. **Storage key** – Project Settings → **API Keys** → copy the **Secret key** (`sb_secret_…`), or under
   "Legacy API keys" the **service_role** key. This is `SUPABASE_SERVICE_ROLE_KEY`. Keep it private.
4. **Project URL** – Project Settings → Data API → `https://<project-ref>.supabase.co`. This is `SUPABASE_URL`.

You do **not** need to create tables or the bucket by hand – the app creates them, enables row-level security,
and creates a public `aloa-media` bucket on first start.

## 2. Configure locally and move your existing data

```bash
npm install
cp .env.example .env          # Windows: copy .env.example .env
```
Edit `.env`: set `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, and for local testing
`NODE_ENV=development` and `PUBLIC_URL=http://localhost:3000`.

```bash
npm run check                    # verifies the real Supabase database + storage connection
npm run migrate -- --dry-run     # shows what will be copied, changes nothing
npm run migrate                  # copies data/aloa.db + uploaded images into Supabase
npm start                        # http://localhost:3000/admin – sign in with your existing account
```

`npm run migrate` safety:
- reads the old SQLite file **read-only** and first saves a JSON copy to `data/exports/`;
- copies admins (passwords keep working), all content (drafts + live), published versions, enquiries, media and activity;
- uploads local images to Supabase Storage and updates every link (content, drafts and old versions);
- verifies row counts afterwards;
- **refuses to run if the Supabase database already has data**. `--replace` overwrites it, but first saves a JSON backup of what is there.

Sessions are not copied – everyone simply signs in again. Your original SQLite file is never modified
(a pre-migration copy is also in `data/backup-before-postgres-*.db`).

New install without old data? Skip `migrate` and run `npm run create-admin` instead.

## 3. Deploy to Hostinger (Node.js Web Apps)

1. Push the project to a **private** GitHub repository (`.gitignore` already excludes `.env`, `data/`, `uploads/`,
   exports and `node_modules/`).
2. hPanel → **Websites → Add website → Node.js Apps** → import from GitHub (or upload a ZIP of the project
   *without* `node_modules`, `data`, `uploads` and `.env`).
3. Build settings:

   | Setting | Value |
   |---|---|
   | Framework / preset | Express (or "Other") |
   | Node.js version | 22.x or 24.x |
   | Entry file | `server/index.js` |
   | Install / build command | `npm ci --omit=dev` (or `npm install --omit=dev`) |
   | Start command | `npm start` |

4. **Environment variables** (hPanel → your Node.js app → Settings / Environment variables):

   | Name | Value |
   |---|---|
   | `NODE_ENV` | `production` |
   | `PUBLIC_URL` | `https://www.yourdomain.com` (your real https address, no trailing slash) |
   | `TRUST_PROXY` | `1` |
   | `DATABASE_URL` | Supabase **Session pooler** URI with your password |
   | `SUPABASE_URL` | `https://<project-ref>.supabase.co` |
   | `SUPABASE_SERVICE_ROLE_KEY` | Secret / service_role key |
   | `SUPABASE_BUCKET` | `aloa-media` (optional) |
   | `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM` | Your mailbox (e.g. Hostinger email: `smtp.hostinger.com`, `465`, `true`) |

   **Do not set `PORT`** – Hostinger provides it; the server reads `process.env.PORT` and listens on `0.0.0.0`.
   Optional: `PG_POOL_MAX` (default 5), `SESSION_DAYS` (7), `MAX_UPLOAD_MB` (5).

5. Deploy. The log should show `Listening on 0.0.0.0:<port> (production)` and `Images: Supabase Storage`.
   If a setting is missing, the server stops with a clear **Configuration problems** message instead of
   running with lost uploads or insecure cookies.
6. Open `https://yourdomain.com/healthz` → `{"ok":true}` means the database is reachable. Then sign in at `/admin`.

## 4. Backups and restore

- `npm run export-data` – saves the whole database (admins, content, versions, enquiries, media list, activity) to
  `data/exports/export-postgres-<date>.json`. The file contains password hashes and leads – keep it private.
- `npm run import-data -- <file.json> [--dry-run] [--replace]` – restores an export.
- Leads can also be exported any time from **Enquiries → Export CSV**.
- Supabase also keeps daily backups on paid plans.

## 5. Commands

| Command | Purpose |
|---|---|
| `npm start` | Start the server (production) |
| `npm run dev` | Start with auto-restart while developing |
| `npm run check` | Pre-deployment check of the real configuration |
| `npm run create-admin` | Create an admin or reset a password (`-- --email you@example.com`) |
| `npm run migrate` | Copy the old SQLite data + images into Supabase |
| `npm run export-data` / `import-data` | JSON backup / restore |
| `npm test` | Automated tests (uses a built-in temporary PostgreSQL – never your Supabase data) |

## 6. Using the admin panel

| Menu | What you manage |
|---|---|
| Dashboard | Enquiry totals, new leads, recent enquiries, publish status, activity log, quick links |
| Website Content | All sections (incl. Problem, Golden Ratio, Theory, Final CTA, Footer, Enquiry Form) + version history |
| Hero / Skills / Instructors / Testimonials / Bonuses / Pricing / FAQs | Section editors – add, edit, delete, duplicate, show/hide, drag to reorder |
| Enquiries | Search, filter, mark contacted, notes, bulk actions, delete, CSV export |
| Media Library | Upload, preview, replace (updates everywhere), delete unused images |
| Countdown Settings | Real expiry date or fixed duration; what happens at zero (never restarts by itself) |
| SEO Settings | Title, description, share image, canonical URL, noindex |
| General Settings | Site title, logo, favicon, WhatsApp, enquiry email, button links, social links |
| Admin Profile | Name, email, password, add/remove administrators |

**Ctrl/Cmd + S** saves the current page. Leaving a page with unsaved changes asks for confirmation.

## Forgot your password?
Use **Forgot password** on the sign-in page (needs SMTP), or run `npm run create-admin -- --email you@example.com`
locally (with the production `DATABASE_URL` in `.env`) to set a new password.

## Project structure
```
server/            Express app, PostgreSQL layer, auth, content service, storage, data transfer, routes
views/landing.ejs  Landing page template (your original markup)
public/            style.css, script.js, assets/ (original images)
admin/             Admin panel
scripts/           create-admin, data (migrate/export/import), check
test/              Automated tests (npm test)
_original/         Untouched copies of the three files you provided
```
