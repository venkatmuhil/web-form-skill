---
name: web-forms
description: >
  Build production-ready web forms for any purpose: contact forms, lead generation, job applications,
  requirements gathering, surveys, feedback forms, and more. Use this skill whenever a user asks to
  "create a form", "build a contact page", "add a lead capture", "make a survey", "set up a job
  application form", "collect requirements", "add an inquiry form", or any variation of wanting
  form UI on a website. Also triggers for "integrate Brevo", "integrate Resend", "integrate SendGrid",
  "send form emails", "form submission webhook", "form validation", or "contact page".
  ALSO triggers in Audit Mode when the user wants to inspect or improve a form they already have:
  "review my form", "audit my contact form", "edit existing form", "improve this form",
  "is my form accessible?", "check my form code", "form code review", "make my form look better",
  "fix form UX", "improve form UI", or any variation pointing at an existing form file.
  Covers full stack: structured user interview, UI generation, business-type field presets,
  pill/toggle multi-select UI, client-side validation, WCAG accessibility, submission delivery
  (managed email — Brevo, Resend, SendGrid, Mailgun, Postmark — or no-3rd-party paths: SMTP/Nodemailer,
  database persistence, Slack/Discord/Teams webhooks, generic webhook), consent + privacy compliance,
  and a separate user-initiated production deploy phase (platform env vars, DNS verification, smoke tests).
  Always use this skill before generating any form-related code — even if the request seems simple.
---

# Web Forms Skill

Build polished, accessible, production-ready web forms — starting with a structured interview,
ending with working code and a deployment checklist.

---

## Phase 0 — Intent Triage

**Decide which mode to run before doing anything else.** Two modes share the same rubrics and reference files:

| Mode | Trigger signals | Where to go |
|------|-----------------|-------------|
| **Greenfield** (new form) | "create / build / add / make / set up" + form noun; no existing file referenced | Continue to **Phase 1 — Interview** |
| **Audit & Improve** (existing form) | A file path is mentioned, code is pasted, or the verb is "review / audit / edit / improve / check / fix / polish / refactor" + form noun; phrases like "existing", "already have", "current form", "my contact page" | Jump to **Phase A — Audit & Improve** (below). Skip Phase 1 entirely. |

Ambiguous? Ask one short question: *"Are we building a new form or improving one you already have? If existing, share the file path."* Do not start the new-form interview on an audit request — it is the wrong shape and will frustrate the user.

Greenfield is the default. Audit Mode never runs unless triggered by the signals above.

---

## Phase 1 — Interview

**Always interview the user before generating any code.** The interview shapes field selection,
tone, UI patterns, and email integration. Ask these questions upfront (all at once, not one by one):

| # | Question | Purpose |
|---|----------|---------|
| 1 | What kind of business is this? (consulting, SaaS, agency, e-commerce, nonprofit, etc.) | Maps to field preset; sets vocabulary |
| 2 | Who fills this form? (founder, enterprise buyer, job applicant, student, customer…) | Sets tone and label language |
| 3 | What's the primary CTA? ("Book a call", "Get a quote", "Apply now", "Send message") | Submit button label + success message copy |
| 4 | Any specific fields you need beyond the defaults? Or fields to remove? | Customise the preset |
| 5 | What are your brand colors? (primary accent hex, background hex) | Inject into Tailwind / CSS |
| 6 | How should submissions be delivered? Pick one or combine: **Brevo · Resend · SendGrid · Mailgun · Postmark** (managed email) · **SMTP/Nodemailer** (own mail server) · **Database** (Postgres/Supabase/Prisma) · **Slack/Discord/Teams webhook** · **Generic webhook** (Zapier/Make/n8n) | Routes to the right integration — see `email-services.md` for all options including no-3rd-party paths |
| 7 | What email address should receive notifications? *(skip if delivery = DB-only or chat-only)* | `NOTIFY_EMAIL` env var |
| 8 | What sender name and email should appear on outgoing emails? *(skip if no email path)* | `SENDER_NAME` / `SENDER_EMAIL` env vars |
| 9 | Do you want invisible CAPTCHA (reCAPTCHA v3) for bot protection? Default: **Yes** for public forms | Adds reCAPTCHA v3 to component + API route |
| 10 | Which consents do you need? Default: **Privacy/GDPR**. Optionally: marketing opt-in, terms of service, age confirmation, custom. For each, paste the linked policy URL — or ask me to scaffold a placeholder page. | Generates consent checkboxes + server-side `<key>ConsentedAt` timestamps. See `compliance.md`. |
| 11 | On success: replace the form inline (default) **or** redirect to a thank-you page (e.g. `/thank-you`) — useful for analytics conversion tracking? | Wires `router.push("/thank-you")` instead of the inline success state |

**Skip questions whose answers are already clear from context.** If the user says
"add a contact form to my Next.js SaaS with Resend", skip Q1, Q6, and partially Q2.
Skip Q9 if the form is behind authentication (auto-answer: No). Default **Yes** for all public-facing forms unless the user says otherwise.
Skip Q10 if the form is behind authentication and the auth flow already captured consent. Default **Yes (privacy only)** for all public forms.
Default Q11 to inline replace unless the user mentions analytics, conversion tracking, or a separate thank-you page.

After the interview, confirm the plan before generating:
> "Here's what I'll build: [form type] for [business], fields: [list], submits via [email service],
> styled [brand color]. Ready to generate?"

---

## Phase 2 — Identify Stack & Format

### Output format
- **Next.js (App Router)** → `src/app/contact/page.tsx` + `src/app/api/contact/route.ts`
- **Next.js (Pages Router)** → `pages/contact.tsx` + `pages/api/contact.ts`
- **Plain React** → single `.jsx` / `.tsx` component
- **Plain HTML / static** → single `.html` file with inline `<script>` and `<style>`
- **Vue / Svelte / Angular** → adapt syntax; note any caveats
- **No mention** → default to HTML artifact (inline preview); offer to export as file

### Styling
- **Tailwind mentioned** → Tailwind classes; brand color as arbitrary value `bg-[#HEX]`
- **Bootstrap / MUI / Chakra** → use that framework's components
- **React + no mention** → default to Tailwind (most common pairing)
- **HTML + no mention** → scoped `<style>` block with CSS custom properties (`--color-accent`)

---

## Phase 3 — Select Field Preset

Read `references/form-presets.md` for full field definitions per preset.

### Business type → preset

| Business type | Preset to use |
|---------------|---------------|
| consulting, advisory, professional services | `consulting` |
| SaaS, software, product, startup | `saas` |
| agency, studio, creative, marketing | `agency` |
| job / hiring / careers | `job-application` |
| survey, feedback, NPS, poll | `survey` |
| requirements, brief, project scope, onboarding | `requirements` |
| generic / unclear / any other | `contact` |

 mutually exclusive options | **Single-select pill row** (radio styled pills) |
| Budget / range bands | Single-select pill row with preset bands |
| Open / freeform | `<input>` or `<textarea>` |
| File needed | `<input type="file">` with type/size validation |

### Conditional / Progressive Logic (optional layer — applies to all presets)

Suggest this pattern when:
- A qualifier question early in the form determines which fields are relevant (e.g. "Are you an individual or a company?")
- The form has 6+ fields and not all apply to every user
- The user asks for a "smart", "dynamic", or "adaptive" form

**Three patterns — pick based on complexity:**

| Pattern | When to use | Key trigger field |
|---------|-------------|-------------------|
| **Field show/hide** | 1–3 dependent fields toggle on a single answer | select, radio, pill toggle |
| **Section skip** | A whole fieldset is irrelevant for one answer path | radio / pill (e.g. "Individual vs Business") |
| **Adaptive multi-step** | 3+ steps where step 2+ content changes based on step 1 | any early qualifier |

Read `references/accessibility-patterns.md → Conditional & Progressive Patterns` for full React + HTML code.

**Rules (non-negotiable for all three patterns):**
- Hidden fields must **never** be `required` or included in submission — clear their values and set `disabled` on hide
- Use the `hidden` attribute (not CSS-only `display:none`) so screen readers skip correctly
- On **reveal**: move focus to the first revealed field
- On **hide**: if focus was inside the hidden section, return focus to the trigger element
- React: use a state flag + conditional render (`condition && <Section />`) for most cases; use `aria-hidden` + `disabled` only when you need CSS transitions

**Pill toggle — React/Tailwind:**
```tsx
const OPTIONS = ['Strategy', 'Design', 'Development', 'Marketing'];
const [selected, setSelected] = useState<string[]>([]);

const toggle = (item: string) =>
  setSelected(prev =>
    prev.includes(item) ? prev.filter(i => i !== item) : [...prev, item]
  );

<div role="group" aria-labelledby="services-label">
  <p id="services-label" className="font-medium mb-2">Services *</p>
  <div className="flex flex-wrap gap-2">
    {OPTIONS.map(opt => (
      <button
        key={opt} type="button"
        onClick={() => toggle(opt)}
        aria-pressed={selected.includes(opt)}
        className={`px-4 py-2 rounded-full border text-sm font-medium transition-colors
          ${selected.includes(opt)
            ? 'bg-[#ACCENT] text-white border-[#ACCENT]'
            : 'bg-white text-gray-700 border-gray-300 hover:border-[#ACCENT]'}`}
      >
        {opt}
      </button>
    ))}
  </div>
</div>
```
Replace `#ACCENT` with brand color from interview.

---

### Conversational / Typeform-style Mode (optional)

Suggest this mode when:
- The form has 5+ questions and a guided, friendly feel matters more than speed
- The audience is consumer-facing or non-technical
- The user says "Typeform", "conversational", "one question at a time", or "quiz-style"

| Feature | Implementation note |
|---------|-------------------|
| One question at a time | Render only the active `<section>`; all others `hidden` + `aria-hidden="true"` |
| Smooth slide transitions | CSS `@keyframes` slide-in/out, or Framer Motion `AnimatePresence` |
| Animated entrance | Slide + fade in from right on advance, left on back |
| Keyboard advance | `Enter` submits current answer and advances; `↑ / ↓` or `← / →` to navigate |
| Progress bar + counter | `role="progressbar"` with `aria-valuenow` / `aria-valuemax`; visible "3 of 8" label |
| Focus management | After transition, focus moves to the input of the newly revealed question |
| Submit placement | Show submit button only on the final question |

Read `references/accessibility-patterns.md → Conversational / Typeform-style Pattern` for full React code.

---

## Phase 4 — Build the Form Component

### React component shell
```tsx
"use client";
import { useState } from "react";

type FormData = { /* typed fields per preset */ };
const INITIAL: FormData = { /* defaults */ };

export default function ContactForm() {
  const [form, setForm]       = useState<FormData>(INITIAL);
  const [loading, setLoading] = useState(false);
  const [error, setError]     = useState<string | null>(null);
  const [submitted, setSubmitted] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true); setError(null);
    try {
      const res = await fetch("/api/contact", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(form),
      });
      if (!res.ok) throw new Error((await res.json()).error ?? "Something went wrong");
      setSubmitted(true);
    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  // Success state — inline replace (default) OR redirect (Q11)
  // If Q11 = thank-you page: use `useRouter().push("/thank-you")` in the try block
  // instead of `setSubmitted(true)`, and drop this block.
  if (submitted) return (
    <div role="status" aria-live="polite" className="text-center py-16">
      <h2 className="text-2xl font-bold mb-2">You're all set!</h2>
      <p className="text-gray-600">/* CTA-specific copy from interview */</p>
    </div>
  );

  return (
    <form onSubmit={handleSubmit} noValidate>
      {error && <p role="alert" className="text-red-600 mb-4">{error}</p>}
      {/* fields */}
      <button type="submit" disabled={loading} aria-busy={loading}>
        {loading ? "Sending…" : "/* CTA label from interview */"}
      </button>
    </form>
  );
}
```

### Accessibility — WCAG 2.1 AA (non-negotiable)
Read `references/accessibility-patterns.md` for complex patterns (file upload, star rating, etc.)

- Every input/textarea/select needs a `<label>` (`for`/`id` pair or `aria-label`)
- Radio/checkbox groups in `<fieldset>` + `<legend>`
- Errors: `role="alert"` or `aria-live="polite"`; never color-only
- Required: `required` attribute + visual asterisk + page legend "* Required"
- Pill toggles: `aria-pressed`; submit button: `aria-busy` while loading
- Focus ring must be visible (never bare `outline: none`)
- On submit with errors: focus first invalid field

### Client-side validation
- HTML5 first: `required`, `type="email"`, `minlength`, `pattern`
- JS: live blur feedback, cross-field checks, custom messages
- Disable submit during loading; re-enable on error

### Consent layer (Phase 1 Q10 — if any consent was chosen)

Read `references/compliance.md → Phase 4` for the full checkbox component.

- One checkbox per consent — never bundle "privacy + marketing" into one
- Required consents use the `required` attribute and are **unchecked** by default (pre-ticked = GDPR violation)
- Policy links open in a new tab with `rel="noopener"`; placed inline within the label
- If the user has no policy page yet, scaffold `/privacy` and/or `/terms` placeholder routes with a "FILL ME IN" body
- Add `<key>Consent: boolean` to `FormData` and `INITIAL` state for each consent

### Security layer (non-negotiable — always add to every form)

Read `references/security-patterns.md` for full copy-paste code for each item.

**Always include in the form component — requires zero visible UI changes:**

1. **Honeypot field** — hidden `<input name="website">` placed just before the submit button. Add `website: ""` to the `FormData` type and `INITIAL` state. See `security-patterns.md → Honeypot`.

2. **Submission timer** — `const [formLoadTime] = useState(() => Date.now());`. Include `_t: formLoadTime` in the JSON body on submit. See `security-patterns.md → Submission time guard`.

3. **reCAPTCHA v3** — include if Q9 = Yes (or for any public form where Q9 was not explicitly No). Wrap the app with `<GoogleReCaptchaProvider>`, call `executeRecaptcha("contact_form")` on submit, include the token as `recaptchaToken` in the body. See `security-patterns.md → reCAPTCHA v3`.

---

## Phase 5 — API Route & Email Integration

Read `references/email-services.md` for all service integrations (Brevo, Resend, SendGrid, etc.)

### Delivery quick-pick

Pick one (or combine — e.g. DB + Slack with no email at all):

| Option | Best for | Free tier | npm package | Reference |
|--------|----------|-----------|-------------|-----------|
| **Brevo** | CRM + email; contact list sync | 300/day | `@getbrevo/brevo` | `email-services.md` |
| **Resend** | Dev-friendly; React Email templates | 100/day | `resend` | `email-services.md` |
| **SendGrid** | Enterprise scale; high deliverability | 100/day | `@sendgrid/mail` | `email-services.md` |
| **Mailgun** | Pay-as-you-go; EU data residency | Trial only | `mailgun.js` | `email-services.md` |
| **Postmark** | Best deliverability; transactional focus | Trial only | `postmark` | `email-services.md` |
| **SMTP / Nodemailer** | Own mail server, Google Workspace, M365, SES SMTP | — | `nodemailer` | `email-services.md → Option 1` |
| **Database persistence** | Store every submission; admin queries; no email needed | — | `@prisma/client` or `@supabase/supabase-js` or `pg` | `email-services.md → Option 2` |
| **Slack / Discord / Teams** | Team in chat; no inbox | — | — (webhook URL) | `email-services.md → Option 3` |
| **Generic webhook** | Zapier / Make / n8n / own backend | — | — | `email-services.md → Option 4` |

### Every API route does these things (in order)

0. **Origin check** — read `Origin` header; if `ALLOWED_ORIGINS` is set and the header is not in the comma-separated allow-list, return `403 Forbidden`. Skip in dev (no `ALLOWED_ORIGINS` set).
1. **Body size guard** — reject raw request body > 50 KB before JSON.parse
2. **Sanitize inputs** — `sanitizeInput()` on name (100), email (254), message (5000), all text fields
3. **Validate required fields** — return `400` if name/email missing or email fails regex
4. **Validate required consents** — return `400` if a required consent (Q10) is false; capture `<key>ConsentedAt` timestamps for the truthy ones. See `compliance.md → Phase 5`.
5. **Bot guards** — all silent `200 { success: true }` on trigger (never tip off bots):
   - Rate limit by IP (in-memory default; **required upgrade to Upstash on serverless**)
   - Honeypot check (`body.website` non-empty)
   - Submission time guard (`body._t` arrived in < 3 seconds)
   - Duplicate email check (same email within 5 minutes)
   - Spam keyword filter
   - reCAPTCHA v3 score check (score < 0.5) — if Q9 = Yes
6. **Deliver the submission** — pick the path(s) chosen in Q6:
   - Email paths → notify team + confirm submitter; `escapeHtml()` every `${value}` in HTML bodies; `sanitizeInput()` in subjects
   - DB path → insert row with sanitized fields, `ipHash`, consent timestamps
   - Chat/webhook path → POST to channel/webhook with retry on 429
7. **Sync contact** (Brevo only, if also using Brevo) — `createContact` with `updateEnabled: true` → upsert into CRM list
8. **Log real failures** — `console.error` on email-service 5xx, DB write failure, webhook non-2xx so platform log drains catch them. Bot-guard short-circuits remain silent.

Read `references/security-patterns.md` for copy-paste implementations of all guards and utilities used in steps 1–6.

**Forms with file upload fields** must submit as `multipart/form-data`, not JSON.
The API route reads the file with `await req.formData()` and attaches it to the notification email.
See `references/email-services.md → File Attachments` for per-service attachment syntax and size limits.

### Environment variables (always list after generating)

Only list the variables that match the chosen delivery option(s) from Q6.

| Variable | Description | When |
|----------|-------------|------|
| `[SERVICE]_API_KEY` | e.g. `BREVO_API_KEY`, `RESEND_API_KEY`, `SENDGRID_API_KEY` | Managed email service |
| `SENDER_NAME` | Display name for outgoing emails | Any email path |
| `SENDER_EMAIL` | Verified sender address (must match verified domain) | Any email path |
| `NOTIFY_EMAIL` | Team inbox for submission notifications | Any email path |
| `BREVO_LIST_ID` | Brevo list ID for contact upsert | Brevo only |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` / `SMTP_SECURE` | SMTP relay credentials | SMTP/Nodemailer |
| `DATABASE_URL` | Postgres connection string | DB path (Prisma / `pg`) |
| `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY` | Supabase project + server-only key | DB path (Supabase) |
| `IP_SALT` | Random hex string for hashing IPs before storage | DB path |
| `SLACK_WEBHOOK_URL` / `DISCORD_WEBHOOK_URL` / `TEAMS_WEBHOOK_URL` | Incoming-webhook URL for chat notifications | Chat path |
| `WEBHOOK_URL` | Outbound webhook destination (Zapier / Make / own backend) | Generic webhook |
| `ALLOWED_ORIGINS` | Comma-separated allowed origins e.g. `https://yourdomain.com` — enforced by step 0 of API contract | All (production) |
| `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` | reCAPTCHA v3 site key — browser-visible | Q9 = Yes |
| `RECAPTCHA_SECRET_KEY` | reCAPTCHA v3 secret — server only | Q9 = Yes |

---

## Phase 6 — Post-Generation & Local Verification

This phase runs **immediately after code generation**. It covers files, install, env vars, and local testing only. **Do not run any deployment steps here** — Deploy is Phase 8, and only runs when the user explicitly asks to deploy.

Always output this block after generating files:

```
✅ Files generated:
   src/app/contact/page.tsx
   src/app/api/contact/route.ts

📦 Install dependency:
   npm install [package]          ← from service table above

🔑 Add to .env.local:
   SENDER_NAME="Your Company"
   SENDER_EMAIL="hello@yourdomain.com"
   NOTIFY_EMAIL="team@yourdomain.com"
   [SERVICE]_API_KEY="your_key_here"
   # BREVO_LIST_ID=3                              ← Brevo only
   ALLOWED_ORIGINS="https://yourdomain.com"
   # NEXT_PUBLIC_RECAPTCHA_SITE_KEY="6Le..."      ← Q9 = Yes
   # RECAPTCHA_SECRET_KEY="6Le..."                ← Q9 = Yes

🌐 Email service setup:
   1. Create account at [service URL]
   2. Get API key → Settings → API Keys
   3. Verify sender domain (add DNS records they provide)
   4. [Brevo] Create contact list → note the list ID

🔒 Security defaults included:
   ✅ escapeHtml — XSS prevention in email HTML
   ✅ sanitizeInput — CRLF strip + length caps
   ✅ Email regex validation + body size guard (50 KB)
   ✅ Honeypot field + submission time guard (3s minimum)
   ✅ In-memory rate limit (5 req / 60s / IP)
   ✅ Duplicate email guard (5-min window) + spam keyword filter
   [ ] reCAPTCHA v3 — set NEXT_PUBLIC_RECAPTCHA_SITE_KEY + RECAPTCHA_SECRET_KEY
       npm install react-google-recaptcha-v3
   [ ] Upgrade rate limiter to Upstash for multi-instance / serverless
       npm install @upstash/ratelimit @upstash/redis

🛡️  Consent + privacy (if Q10 ≠ none):
   ✅ Required consents unchecked by default; each has its own checkbox + dedicated policy link
   ✅ Server records <key>ConsentedAt timestamps alongside the submission
   [ ] Privacy policy page filled in: /privacy
   [ ] Terms page filled in (if applicable): /terms
   [ ] Data retention policy decided — default 90-day purge (see compliance.md)

🧪 Test locally:
   npm run dev → fill form → check destination (inbox / DB row / chat channel)
   npm run build → confirm 0 TypeScript/build errors
   Bot-guard sanity check: submit within 3s OR with honeypot filled → success page shows
     but NOTHING arrives in destination (silent 200 working)

🚀 Ready to deploy?
   When local tests pass and you're ready to ship, say "deploy the form"
   and I'll walk you through Phase 8 (platform env, DNS verification, smoke test).
   Don't paste production secrets or DNS records until you're ready — that's a
   separate phase, not part of generation.
```

---

## Phase 7 — Multi-step Forms (optional)

Suggest multi-step only when: user asks, OR the form has 8+ fields,
OR the form type is `requirements` / `job-application`.

Pattern: `Contact Info → Project Details → Review & Submit`
- Validate current step's fields before `Next`; disable button if invalid
- `role="progressbar"` with `aria-valuenow` / `aria-valuemax`
- Back button must preserve entered values (keep single state object)
- Submit fires only on final step
- See `references/accessibility-patterns.md` for full multi-step pattern

---

## Phase 8 — Deploy (user-initiated only)

**Activation rule — strict.** This phase runs only when **both** are true:
1. The user has confirmed they tested the form locally (real submission landed in the destination, bot guards behave correctly).
2. The user has explicitly said something like "deploy", "ship it", "go live", "push to prod".

Do **not** start the Deploy walkthrough as part of initial generation. Phase 6 ends with a one-line prompt inviting the user to ask for Deploy — wait for them to take that step. Production arrangements (DNS, platform secret stores, smoke tests, monitoring) make no sense while the form is still a concept being iterated on.

Read `references/deployment.md` and walk through it step by step:

1. **Pre-flight check** — build clean, local submission worked, `.env.local` is gitignored
2. **Pick target platform** — Vercel / Netlify / Cloudflare Pages / Railway / Fly / Render / Docker-VPS; identify if it's serverless (rate limiter must move to Upstash)
3. **Set env vars** on the platform — every `*_API_KEY`, `*_PASS`, `*_SECRET`, `*_WEBHOOK_URL`, `DATABASE_URL`, `IP_SALT`, `SUPABASE_SERVICE_ROLE_KEY` must be marked secret/encrypted. Only `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` is browser-visible.
4. **Verify sender domain** (email paths only) — SPF + DKIM + DMARC records resolve via `dig` or MXToolbox before first send. Without these, mail lands in spam regardless of code quality.
5. **Confirm origin allow-list** — `ALLOWED_ORIGINS` set to the live URL(s); the route's step-0 origin check enforces it.
6. **Production smoke test** — real submission lands in destination; bot path returns silent 200 with nothing delivered; rate limit fires by request 6.
7. **Lighthouse + axe a11y pass** on the deployed URL — target ≥ 95.
8. **Wire monitoring** — surface platform logs for the API route; optional Sentry/Axiom hookup.
9. **Hygiene** — confirm `.env.local` never committed; rotate any key ever pasted in chat; schedule retention purge (if DB).

Stop and confirm with the user at the end of each step. The Deploy phase typically takes 20–45 minutes spread over a working session — it is not a one-shot tool call.

---

## Phase A — Audit & Improve (existing form)

**Activation:** entered from Phase 0 when the user wants to review, edit, or improve a form
that already exists. Replaces Phases 1–5 for this run. Phases 6–8 (verification, deploy)
may still run after fixes are applied.

The same reference files that drive generation drive the audit — there is no separate
rubric document. Standards stay in lockstep across both modes.

### A.1 — Locate the form

Ask for paths if not given. Read all relevant files with `Read`:
- Form component (`*.tsx` / `*.jsx` / `*.vue` / `*.html`)
- API route (`route.ts` / `[name].ts` / server handler)
- Env example (`.env.example`) and any config touching delivery

Do not guess — if the user only gave one file but the framework implies a counterpart
(Next.js page without route, or vice versa), ask before assuming it's missing.

### A.2 — Run the checklist audit

For each category, mark **✅ pass** / **⚠️ partial** / **❌ missing** with `file:line`
evidence. Reuse the existing reference files as the rubric — never duplicate rules into
the audit output.

| Category | Rubric source | Required spot-checks |
|----------|---------------|----------------------|
| Security | `references/security-patterns.md` | `escapeHtml` on all `${value}` in email HTML, `sanitizeInput` on text fields, body size guard (50 KB), honeypot field, submission time guard (3s), rate limiter, duplicate-email guard, spam keyword filter, `ALLOWED_ORIGINS` step-0 check, reCAPTCHA v3 if public |
| Accessibility | `references/accessibility-patterns.md` | every input has `<label for>` / `aria-label`, radio/checkbox groups wrapped in `<fieldset><legend>`, errors use `role="alert"` or `aria-live`, errors are not color-only, `required` attribute + visual asterisk + page legend, submit button `aria-busy` while loading, visible focus ring (no bare `outline:none`), focus moves to first invalid field on submit |
| Validation | SKILL.md Phase 4 | HTML5 attrs (`required`, `type=email`, `minlength`, `pattern`), server-side regex + length caps, submit disabled during loading, cross-field rules where relevant |
| Delivery & API contract | `references/email-services.md` + SKILL.md Phase 5 nine-step list | all nine steps present and in order (origin → size → sanitize → validate → consents → bot guards → deliver → CRM upsert if any → log real failures); bot-guard paths return silent `200`; HTML email bodies escape every interpolated value |
| Compliance | `references/compliance.md` | one checkbox per consent (no bundling), required consents unchecked by default, server records `<key>ConsentedAt` timestamps, policy links `target="_blank" rel="noopener"`, placeholder `/privacy` and `/terms` routes exist if linked |
| UI / UX polish | `references/accessibility-patterns.md → UI/UX heuristics for form polish` | visual hierarchy, spacing rhythm, inline error placement, ≥ 44px tap targets, `inputmode` + `autocomplete`, brand accent applied to CTA + focus, microcopy, motion respects `prefers-reduced-motion` |

**Severity buckets for the report:**
- **P0** — security or correctness (XSS, missing sanitization, no rate limit, missing origin check, broken submission)
- **P1** — accessibility violations (WCAG 2.1 AA) and required-consent gaps
- **P2** — UI/UX polish (spacing, hierarchy, tap targets, microcopy)
- **P3** — nice-to-have (motion, advanced patterns, conversational mode)

**Output format:** a single Markdown table of findings, each row with category, severity,
file:line, what's wrong, and the one-line fix. End with a one-line summary:
`Found X P0, Y P1, Z P2, W P3 issues.`

### A.3 — Present findings, ask before fixing

Do **not** edit anything until the user picks a scope. Offer:

1. **Fix P0 + P1** (security + a11y + correctness) — the safe, high-value default
2. **Fix everything** including UI/UX polish
3. **Fix specific items** — user names rows from the report
4. **Just leave me the report** — no edits

This gating mirrors the Deploy phase: the audit is cheap, the fix is not.

### A.4 — Apply fixes

When the user confirms, use `Edit` on their existing files. Rules:

- **Preserve their stack** — do not swap Tailwind for CSS-in-JS, do not move from Pages to
  App Router, do not rename their variables.
- **Reuse the generator's snippets verbatim** — pull from `security-patterns.md`,
  `accessibility-patterns.md`, `compliance.md`. The audit fix and a freshly-generated
  form should produce byte-identical security utilities.
- **Smallest diff that satisfies the rule** — do not "improve while you're in there".
  Each edit traces back to one numbered finding from A.2.
- **Group related edits** — all security fixes in one pass, all a11y fixes in another,
  so the diff stays reviewable.

After edits, output:
- A short diff summary (file → list of finding IDs fixed)
- The same `🧪 Test locally` block from Phase 6 (adapted to the user's framework)
- Offer Phase 8 (Deploy) only if the user explicitly asks

### A.5 — When to fall back to greenfield

If the audit finds the existing form is so far from the rubric that patching is more work
than rewriting (e.g. no API route at all, wrong framework version, mixes server and client
state incorrectly), say so explicitly and offer: *"This is closer to a rewrite than a fix
— want me to run the Phase 1 interview and regenerate cleanly?"* Let the user decide.

---

## Reference Files

| File | When to read |
|------|--------------|
| `references/form-presets.md` | Phase 3 — always; field definitions per business type |
| `references/email-services.md` | Phase 5 — managed services (Brevo/Resend/SendGrid/Mailgun/Postmark) **and** no-3rd-party paths (SMTP/Nodemailer, DB persistence, Slack/Discord/Teams, generic webhook); file attachment syntax |
| `references/accessibility-patterns.md` | Pills, star ratings, file upload, conditional/progressive patterns, Typeform-style, multi-step; **UI/UX polish heuristics** used by Phase A |
| `references/security-patterns.md` | Phase 4 + Phase 5 — always; all security utilities (escapeHtml, sanitizeInput, rate limiter, honeypot, time guard, duplicate detection, spam filter, reCAPTCHA v3, origin guard) |
| `references/compliance.md` | Phase 1 Q10 + Phase 4 (consent UI) + Phase 5 (consent timestamping) + Phase 8 (data retention, IP hashing, RTBF) |
| `references/deployment.md` | Phase 8 only — never during generation. Platform env vars, DNS verification (SPF/DKIM/DMARC), production smoke test, serverless rate-limit upgrade, monitoring |
