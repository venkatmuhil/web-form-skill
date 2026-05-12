---
name: web-forms
description: >
  Build or audit production-ready web forms — contact, lead capture, job application,
  requirements gathering, survey, feedback. Triggers on "create / build / add / make a form",
  "contact page", "lead capture", "survey", "job application", "form validation",
  "send form emails", "integrate Brevo / Resend / SendGrid / Mailgun / Postmark", "form webhook".
  Also triggers in audit mode: "review / audit / edit / improve / fix my form", "form code review",
  "make my form accessible", "is my form secure". Covers interview, UI generation, presets, pill /
  toggle multi-select, validation, WCAG accessibility, delivery (managed email, SMTP, DB, chat /
  generic webhook), consent + privacy compliance, and a separate user-initiated deploy phase.
  Always use this skill before generating any form-related code.
---

# Web Forms Skill

Build polished, accessible, production-ready web forms — starting with a structured interview,
ending with working code and a deployment checklist.

---

## Phase 1 — Triage

Decide which mode to run before doing anything else.

| Mode | Trigger signals | Where to go |
|------|-----------------|-------------|
| **Greenfield** (default) | "create / build / add / make / set up" + form noun; no existing file referenced | Phase 2 |
| **Audit & Improve** | A file path is mentioned, code is pasted, or the verb is "review / audit / edit / improve / check / fix / polish / refactor" + form noun; phrases like "existing", "already have", "current form" | Phase 11 (skip Phases 2–7) |

Ambiguous? Ask one short question: *"Are we building a new form or improving one you already have? If existing, share the file path."*

---

## Phase 2 — Interview

**Always interview before generating any code.** Ask all questions at once, not one by one.

| # | Question | Purpose |
|---|----------|---------|
| 1 | What kind of business is this? (consulting, SaaS, agency, e-commerce, nonprofit…) | Maps to field preset; sets vocabulary |
| 2 | Who fills this form? (founder, enterprise buyer, applicant, customer…) | Tone + label language |
| 3 | What's the primary CTA? ("Book a call", "Get a quote", "Apply now", "Send message") | Submit-button label + success copy |
| 4 | Any specific fields you need beyond the defaults? Or fields to remove? | Customise the preset |
| 5 | Brand colors? (primary accent hex, background hex) — *skip if Phase 4 detected design tokens* | Inject into Tailwind / CSS |
| 6 | How should submissions be delivered? **Brevo · Resend · SendGrid · Mailgun · Postmark** · **SMTP / Nodemailer** · **Database** · **Slack / Discord / Teams** · **Generic webhook** (combinable) | Routes to the right integration |
| 7 | What email address should receive notifications? *(skip if DB-only or chat-only)* | `NOTIFY_EMAIL` |
| 8 | Sender name and email on outgoing mail? *(skip if no email path)* | `SENDER_NAME` / `SENDER_EMAIL` |
| 9 | Invisible CAPTCHA (reCAPTCHA v3)? Default **Yes** for public forms | Adds reCAPTCHA v3 to component + route |
| 10 | Which consents? Default **Privacy/GDPR**. Optionally marketing opt-in, ToS, age confirmation, custom. Paste policy URL or ask me to scaffold a placeholder. | Generates consent checkboxes + server `<key>ConsentedAt` timestamps |
| 11 | On success: replace inline (default) or redirect to a thank-you page (e.g. `/thank-you`)? | Wires `router.push(...)` instead of inline success |
| 12 | *(DOB field only)* Minimum age? **13** (COPPA), **16** (GDPR-K), **18**, **21**, or **none** | Sets `max=` + server age check |

**Skips.** Skip a question whose answer is clear from context (e.g. "Next.js SaaS with Resend" → skip 1, 6, partly 2). Skip Q9 + Q10 if the form is behind auth. Default Q11 to inline replace unless the user mentions analytics or conversion tracking.

After the interview, confirm before generating:
> "Here's what I'll build: [form type] for [business], fields: [list], submits via [service], styled [brand color]. Ready to generate?"

---

## Phase 3 — Stack, format, styling

| Signal | Output format | Default styling |
|--------|--------------|----------------|
| Next.js App Router | `src/app/contact/page.tsx` + `src/app/api/contact/route.ts` | Tailwind |
| Next.js Pages Router | `pages/contact.tsx` + `pages/api/contact.ts` | Tailwind |
| Plain React | single `.tsx` component | Tailwind (most common pairing) |
| Plain HTML / static | single `.html` with inline `<script>` / `<style>` | scoped `<style>` + CSS custom properties |
| Vue / Svelte / Angular | adapt syntax; note caveats | framework's idiomatic style |
| No mention | HTML artifact (inline preview); offer file export | scoped `<style>` |

If the user names Tailwind, Bootstrap, MUI, or Chakra explicitly, use that — brand color goes in as an arbitrary value (`bg-[#HEX]`) until Phase 4 says otherwise.

---

## Phase 4 — Design-context probe

Read [`references/design-context.md`](references/design-context.md) and run a **bounded, read-only** probe before generating. Skip only for standalone HTML files with no build step.

Stop at the first hit per category:

1. **Design doc** — `design.md` / `DESIGN.md` / `docs/design.md` / `docs/design-system.md`
2. **Token sources** — `tailwind.config.{js,ts,mjs,cjs}` (colors, radii, fonts), then `tokens.json` / `design-tokens.json`, then CSS custom properties in `globals.css`
3. **Component primitives** — `components/ui/{button,input,label,textarea}.tsx` (shadcn); `@chakra-ui/react`, `@mui/material`, `react-aria-components` in `package.json`; or custom `components/Form*.tsx`
4. **Dark mode** — any `dark:` variant or `[data-theme=dark]` selector

**Output one line before Phase 5:**

```
🎨 Design context: [tailwind: primary=#xxx, radius=lg] + [shadcn: Button, Input, Label]
   — generated form will use these.
```

Or, if nothing matched: `🎨 Design context: none detected — using brand color from Q5.`

**Effect:** Q5 is skipped when tokens were detected. Phase 6 uses `bg-primary` instead of `bg-[#HEX]` and imports detected primitives. Phase 11 audits against the same probe.

---

## Phase 5 — Field preset + UI patterns

Read [`references/form-presets.md`](references/form-presets.md) for full field definitions per preset.

| Business type | Preset |
|---------------|--------|
| consulting, advisory, professional services | `consulting` |
| SaaS, software, product, startup | `saas` |
| agency, studio, creative, marketing | `agency` |
| job / hiring / careers | `job-application` |
| survey, feedback, NPS, poll | `survey` |
| requirements, brief, project scope, onboarding | `requirements` |
| generic / unclear / any other | `contact` |

### UI pattern per field type

| Field shape | UI |
|-------------|-----|
| 4+ predefined options | **Multi-select pill toggles** |
| 2–3 mutually exclusive options | **Single-select pill row** (radio styled pills) |
| Budget / range bands | Single-select pill row with preset bands |
| Open / freeform | `<input>` or `<textarea>` |
| File needed | `<input type="file">` with type / size validation |

Pill-toggle code: [`accessibility-patterns.md → Pill toggles`](references/accessibility-patterns.md).

### Conditional / progressive logic (optional)

Suggest when an early qualifier determines which fields are relevant, the form has 6+ fields not all of which apply, or the user asks for a "smart / dynamic / adaptive" form.

| Pattern | When | Trigger field |
|---------|------|--------------|
| **Field show/hide** | 1–3 dependent fields toggle on a single answer | select, radio, pill toggle |
| **Section skip** | A whole fieldset is irrelevant for one path | radio / pill |
| **Adaptive multi-step** | 3+ steps where step 2+ changes based on step 1 | any early qualifier |

Full code: [`accessibility-patterns.md → Conditional & Progressive Patterns`](references/accessibility-patterns.md).

**Non-negotiable rules** (all three patterns):
- Hidden fields must **never** be `required` or submitted — clear values, set `disabled` on hide
- Use the `hidden` attribute (not CSS `display:none`) so screen readers skip
- On reveal: focus the first revealed field. On hide: if focus was inside, return focus to the trigger.

### Conversational / Typeform-style (optional)

Suggest when 5+ questions and a guided feel matters, the audience is consumer-facing, or the user says "Typeform", "conversational", "one question at a time", or "quiz-style". Full code + a11y notes: [`accessibility-patterns.md → Conversational / Typeform-style Pattern`](references/accessibility-patterns.md).

---

## Phase 6 — Build the component

Use the React component shell in [`accessibility-patterns.md → Component shell`](references/accessibility-patterns.md). Replace the typed fields and CTA copy with values from the interview.

### Accessibility — WCAG 2.1 AA (non-negotiable)

- Every input / textarea / select needs a `<label>` (`for`/`id` or `aria-label`)
- Radio / checkbox groups in `<fieldset>` + `<legend>`
- Errors: `role="alert"` or `aria-live="polite"`; never color-only
- `required` attribute + visual asterisk + page legend "* Required"
- Pill toggles: `aria-pressed`; submit button: `aria-busy` while loading
- Focus ring must be visible (never bare `outline: none`)
- On submit with errors: focus the first invalid field

Complex patterns (file upload, star rating, multi-step, etc.): [`accessibility-patterns.md`](references/accessibility-patterns.md).

### Client-side validation

Read [`references/field-validation.md`](references/field-validation.md) for per-type snippets (phone, email, URL, DOB) — same source drives Greenfield and Audit.

- HTML5 first: `required`, correct `type`, `minlength`, `pattern`, `inputmode`, `autocomplete`
- Validate **on blur**, not on every keystroke; re-validate live only after a field has been blurred once and shown an error
- Specific, actionable error copy ("Email must include @"); never just "Invalid"
- Pair color with icon + text (WCAG 1.4.1)
- **Never disable submit.** Let the user submit, surface errors, focus the first invalid field. Disable only while `loading`.
- Mobile: input font-size ≥ 16 px (avoids iOS zoom); tap target ≥ 44 × 44 px

### Consent layer (if Q10 ≠ none)

Read [`references/compliance.md → Phase 4`](references/compliance.md) for the full checkbox component.

- One checkbox per consent — never bundle privacy + marketing into one
- Required consents use `required` and are **unchecked by default** (pre-ticked = GDPR violation)
- Policy links open in a new tab with `rel="noopener"`; inline within the label
- If no policy page exists, scaffold `/privacy` and/or `/terms` placeholder routes
- Add `<key>Consent: boolean` to `FormData` and `INITIAL` for each consent

### Security layer (non-negotiable — always include)

Read [`references/security-patterns.md`](references/security-patterns.md) for copy-paste code.

1. **Honeypot** — hidden `<input name="website">` just before submit. Add `website: ""` to `FormData` / `INITIAL`.
2. **Submission timer** — `const [formLoadTime] = useState(() => Date.now());`. Send `_t: formLoadTime` in the JSON body.
3. **reCAPTCHA v3** — if Q9 = Yes. Wrap with `<GoogleReCaptchaProvider>`, call `executeRecaptcha("contact_form")` on submit, send the token as `recaptchaToken`.

---

## Phase 7 — API route & delivery

Follow [`references/api-contract.md`](references/api-contract.md): every route runs the same nine steps in order (origin → size → sanitize → validate → consents → bot guards → deliver → CRM upsert → log), with the delivery quick-pick table and env-var list at the bottom.

Per-service integration code (Brevo, Resend, SendGrid, Mailgun, Postmark, SMTP/Nodemailer, DB, chat/webhook): [`references/email-services.md`](references/email-services.md).

---

## Phase 8 — Local verification

Runs immediately after code generation. Covers files, install, env vars, and local testing only. **Do not run deployment steps here** — Deploy is Phase 10 and only runs when the user explicitly asks.

Always output this block after generating files. **Omit lines that don't apply to the chosen delivery (e.g. drop Brevo lines if delivery is Resend; drop reCAPTCHA lines if Q9 = No).**

```
✅ Files generated:
   src/app/contact/page.tsx
   src/app/api/contact/route.ts

📦 Install dependency:
   npm install [package]          ← from delivery quick-pick

🔑 Add to .env.local:
   SENDER_NAME="Your Company"
   SENDER_EMAIL="hello@yourdomain.com"
   NOTIFY_EMAIL="team@yourdomain.com"
   [SERVICE]_API_KEY="your_key_here"
   BREVO_LIST_ID=3
   ALLOWED_ORIGINS="https://yourdomain.com"
   NEXT_PUBLIC_RECAPTCHA_SITE_KEY="6Le..."
   RECAPTCHA_SECRET_KEY="6Le..."

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
   ✅ Required consents unchecked by default; one checkbox per consent
   ✅ Server records <key>ConsentedAt timestamps
   [ ] Privacy policy page filled in: /privacy
   [ ] Terms page filled in (if applicable): /terms
   [ ] Data retention policy decided — default 90-day purge (see compliance.md)

🧪 Test locally:
   npm run dev → fill form → check destination (inbox / DB row / chat channel)
   npm run build → confirm 0 TypeScript/build errors
   Bot-guard sanity check: submit within 3s OR with honeypot filled → success page shows
     but NOTHING arrives in destination (silent 200 working)

🚀 Ready to deploy?
   When local tests pass, say "deploy the form" and I'll walk you through Phase 10.
   Don't paste production secrets or DNS records until then — that's a separate phase.
```

---

## Phase 9 — Multi-step forms (optional)

Suggest multi-step only when: the user asks, the form has 8+ fields, or the preset is `requirements` / `job-application`.

Pattern: `Contact Info → Project Details → Review & Submit`.

- Validate current step's fields before `Next`; disable the button if invalid
- `role="progressbar"` with `aria-valuenow` / `aria-valuemax`
- Back button preserves entered values (single state object)
- Submit fires only on the final step

Full multi-step pattern: [`accessibility-patterns.md`](references/accessibility-patterns.md).

---

## Phase 10 — Deploy (user-initiated only)

**Activation — strict.** Runs only when **both** are true:
1. The user has confirmed they tested locally (real submission landed; bot guards behave).
2. The user explicitly said "deploy", "ship it", "go live", "push to prod".

Do not start the Deploy walkthrough as part of initial generation. Phase 8 ends with a one-line prompt inviting the user to ask for Deploy — wait for them to take that step.

Read [`references/deployment.md`](references/deployment.md) and walk through it step by step:

1. **Pre-flight** — clean build, local submission worked, `.env.local` gitignored
2. **Pick target platform** — Vercel / Netlify / Cloudflare Pages / Railway / Fly / Render / Docker-VPS; identify serverless (rate limiter must move to Upstash)
3. **Set env vars** — every `*_API_KEY`, `*_PASS`, `*_SECRET`, `*_WEBHOOK_URL`, `DATABASE_URL`, `IP_SALT`, `SUPABASE_SERVICE_ROLE_KEY` marked secret. Only `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` is browser-visible.
4. **Verify sender domain** (email paths) — SPF + DKIM + DMARC resolve via `dig` or MXToolbox before first send
5. **Confirm origin allow-list** — `ALLOWED_ORIGINS` set to the live URL(s)
6. **Production smoke test** — real submission lands; bot path returns silent 200; rate limit fires by request 6
7. **Lighthouse + axe a11y pass** on the deployed URL — target ≥ 95
8. **Wire monitoring** — surface platform logs for the API route; optional Sentry / Axiom
9. **Hygiene** — `.env.local` never committed; rotate any key ever pasted in chat; schedule retention purge (if DB)

Stop and confirm at the end of each step. Deploy typically spans 20–45 minutes over a working session — not a one-shot tool call.

---

## Phase 11 — Audit & Improve (existing form)

Entered from Phase 1 when the user wants to review, edit, or improve a form that already exists. Replaces Phases 2–7 for this run. Phases 8 and 10 may still run after fixes are applied.

### 11.1 — Locate the form

Ask for paths if not given. Read all relevant files: form component, API route, env example, any config touching delivery. If the user gave only one file but the framework implies a counterpart (Next.js page without route, or vice versa), ask before assuming it's missing.

### 11.2 — Run the audit

Walk every category in [`references/audit-rubric.md`](references/audit-rubric.md). Mark each row **✅ pass** / **⚠️ partial** / **❌ missing** with `file:line` evidence. The same reference files that drive generation drive the audit — never duplicate rules into the audit output.

**Output format:** a single Markdown table of findings, each row with category, severity (P0–P3 per `audit-rubric.md`), `file:line`, what's wrong, the one-line fix. End with:
`Found X P0, Y P1, Z P2, W P3 issues.`

### 11.3 — Present findings, ask before fixing

Do **not** edit anything until the user picks a scope:

1. **Fix P0 + P1** (security + a11y + correctness) — the safe, high-value default
2. **Fix everything** including UI / UX polish
3. **Fix specific items** — user names rows from the report
4. **Just leave me the report** — no edits

### 11.4 — Apply fixes

When the user confirms, use `Edit` on their existing files:

- **Preserve their stack** — don't swap Tailwind for CSS-in-JS, don't move Pages ↔ App Router, don't rename their variables
- **Reuse the generator's snippets verbatim** — pull from `security-patterns.md`, `accessibility-patterns.md`, `compliance.md`. Audit fixes and freshly generated forms produce byte-identical security utilities.
- **Smallest diff that satisfies the rule** — no "improve while you're in there". Each edit traces back to one finding from 11.2.
- **Group related edits** — security fixes in one pass, a11y in another, so the diff stays reviewable

After edits, output: a diff summary (file → finding IDs fixed), the `🧪 Test locally` block adapted to their framework, and offer Phase 10 only if the user explicitly asks.

### 11.5 — Fall back to greenfield

If patching is more work than rewriting (no API route at all, wrong framework version, mixed client / server state), say so and offer: *"This is closer to a rewrite than a fix — want me to run the Phase 2 interview and regenerate cleanly?"* Let the user decide.

---

## Reference files

| File | When to read |
|------|--------------|
| [`form-presets.md`](references/form-presets.md) | Phase 5 |
| [`design-context.md`](references/design-context.md) | Phase 4 + Phase 11 |
| [`accessibility-patterns.md`](references/accessibility-patterns.md) | Phase 5, 6, 9; UI/UX polish in Phase 11 |
| [`field-validation.md`](references/field-validation.md) | Phase 6 + Phase 7 + Phase 11 |
| [`security-patterns.md`](references/security-patterns.md) | Phase 6 + Phase 7 + Phase 11 (always) |
| [`compliance.md`](references/compliance.md) | Phase 2 Q10 + Phase 6 + Phase 7 + Phase 10 |
| [`api-contract.md`](references/api-contract.md) | Phase 7 + Phase 11 (delivery audit) |
| [`email-services.md`](references/email-services.md) | Phase 7 (per-service code, file attachments) |
| [`audit-rubric.md`](references/audit-rubric.md) | Phase 11 |
| [`deployment.md`](references/deployment.md) | Phase 10 only |
