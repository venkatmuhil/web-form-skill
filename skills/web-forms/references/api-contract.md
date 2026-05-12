# API Route Contract

Every form's API route runs the same nine steps in the same order. This
file is the single source of truth — both the build phase and the audit
phase reference it.

Read [`security-patterns.md`](security-patterns.md) for the copy-paste
implementation of each guard and utility named below.

---

## The nine steps (in order)

0. **Origin check** — read the `Origin` header. If `ALLOWED_ORIGINS` is set
   and the header is not in the comma-separated allow-list, return
   `403 Forbidden`. Skip when `ALLOWED_ORIGINS` is unset (dev).
1. **Body size guard** — reject raw request body > 50 KB before
   `JSON.parse`.
2. **Sanitize inputs** — `sanitizeInput()` on every text field: name
   (cap 100), email (254), message (5000), all other text fields.
3. **Validate required fields per field type** — `400` if any required
   field is missing or any typed field fails its server-side check.
   Email: regex from `security-patterns.md`. Phone, URL, DOB: per-type
   snippets in [`field-validation.md`](field-validation.md)
   (`parsePhoneNumberFromString().isValid()` → store E.164; `new URL()`
   + protocol allow-list; UTC `Date.UTC` age check).
4. **Validate required consents** — `400` if any required consent
   (Phase 2 Q10) is false. Capture `<key>ConsentedAt` timestamps for the
   truthy ones. See [`compliance.md`](compliance.md).
5. **Bot guards** — every trigger returns silent `200 { success: true }`
   so bots can't probe:
   - Rate limit by IP (in-memory default; **upgrade to Upstash on
     serverless**)
   - Honeypot check — `body.website` non-empty
   - Submission time guard — `body._t` arrived < 3 s ago
   - Duplicate email — same email within 5 minutes
   - Spam keyword filter
   - reCAPTCHA v3 score < 0.5 (if Q9 = Yes)
6. **Deliver the submission** — per the path(s) chosen in Q6:
   - **Email paths** — notify team + confirm submitter. `escapeHtml()`
     every `${value}` in HTML bodies; `sanitizeInput()` in subject lines.
   - **DB path** — insert row with sanitized fields, `ipHash`, consent
     timestamps.
   - **Chat / webhook path** — POST to channel/webhook with retry on 429.
7. **Sync contact** *(Brevo only)* — `createContact` with
   `updateEnabled: true` to upsert into the CRM list.
8. **Log real failures** — `console.error` on email-service 5xx, DB write
   failure, webhook non-2xx. Bot-guard short-circuits stay silent so
   platform log drains see only real problems.

**Forms with file upload fields** submit `multipart/form-data` instead of
JSON. The route reads files with `await req.formData()` and attaches
them to the notification email. See
[`email-services.md → File Attachments`](email-services.md) for
per-service attachment syntax and size limits.

---

## Delivery quick-pick

Pick one — or combine (e.g. DB + Slack, no email at all):

| Option | Best for | Free tier | npm package |
|--------|----------|-----------|-------------|
| **Brevo** | CRM + email; contact list sync | 300/day | `@getbrevo/brevo` |
| **Resend** | Dev-friendly; React Email templates | 100/day | `resend` |
| **SendGrid** | Enterprise scale; deliverability | 100/day | `@sendgrid/mail` |
| **Mailgun** | Pay-as-you-go; EU data residency | Trial | `mailgun.js` |
| **Postmark** | Best deliverability; transactional | Trial | `postmark` |
| **SMTP / Nodemailer** | Own mail server, Workspace, M365, SES SMTP | — | `nodemailer` |
| **Database** | Store every submission; admin queries | — | `@prisma/client` / `@supabase/supabase-js` / `pg` |
| **Slack / Discord / Teams** | Team chat; no inbox | — | webhook URL |
| **Generic webhook** | Zapier / Make / n8n / own backend | — | — |

Full integration code per service:
[`email-services.md`](email-services.md).

---

## Environment variables

Only list those that match the chosen delivery path(s).

| Variable | Description | When |
|----------|-------------|------|
| `[SERVICE]_API_KEY` | e.g. `BREVO_API_KEY`, `RESEND_API_KEY`, `SENDGRID_API_KEY` | Managed email service |
| `SENDER_NAME` | Display name for outgoing emails | Any email path |
| `SENDER_EMAIL` | Verified sender address | Any email path |
| `NOTIFY_EMAIL` | Team inbox for submissions | Any email path |
| `BREVO_LIST_ID` | Brevo list ID for contact upsert | Brevo only |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` / `SMTP_SECURE` | SMTP relay credentials | SMTP / Nodemailer |
| `DATABASE_URL` | Postgres connection string | DB (Prisma / `pg`) |
| `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY` | Supabase project + server-only key | DB (Supabase) |
| `IP_SALT` | Random hex for hashing IPs before storage | DB path |
| `SLACK_WEBHOOK_URL` / `DISCORD_WEBHOOK_URL` / `TEAMS_WEBHOOK_URL` | Incoming-webhook URL | Chat path |
| `WEBHOOK_URL` | Outbound webhook destination | Generic webhook |
| `ALLOWED_ORIGINS` | Comma-separated allowed origins | All (production) |
| `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` | reCAPTCHA v3 site key — browser-visible | Q9 = Yes |
| `RECAPTCHA_SECRET_KEY` | reCAPTCHA v3 secret — server only | Q9 = Yes |
