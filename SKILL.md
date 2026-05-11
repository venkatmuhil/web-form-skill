---
name: web-forms
description: >
  Build production-ready web forms for any purpose: contact forms, lead generation, job applications,
  requirements gathering, surveys, feedback forms, and more. Use this skill whenever a user asks to
  "create a form", "build a contact page", "add a lead capture", "make a survey", "set up a job
  application form", "collect requirements", "add an inquiry form", or any variation of wanting
  form UI on a website. Also triggers for "integrate Brevo", "integrate Resend", "integrate SendGrid",
  "send form emails", "form submission webhook", "form validation", or "contact page".
  Covers full stack: structured user interview, UI generation, business-type field presets,
  pill/toggle multi-select UI, client-side validation, WCAG accessibility, email service integration
  (Brevo, Resend, SendGrid, Mailgun, Postmark), and webhook submission.
  Always use this skill before generating any form-related code — even if the request seems simple.
---

# Web Forms Skill

Build polished, accessible, production-ready web forms — starting with a structured interview,
ending with working code and a deployment checklist.

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
| 6 | Which email service? (Brevo · Resend · SendGrid · Mailgun · Postmark · None/webhook) | Routes to the right integration |
| 7 | What email address should receive notifications? | `NOTIFY_EMAIL` env var |
| 8 | What sender name and email should appear on outgoing emails? | `SENDER_NAME` / `SENDER_EMAIL` env vars |

**Skip questions whose answers are already clear from context.** If the user says
"add a contact form to my Next.js SaaS with Resend", skip Q1, Q6, and partially Q2.

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

  // Success state REPLACES the form entirely
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

---

## Phase 5 — API Route & Email Integration

Read `references/email-services.md` for all service integrations (Brevo, Resend, SendGrid, etc.)

### Service quick-pick

| Service | Best for | Free tier | npm package |
|---------|----------|-----------|-------------|
| **Brevo** | CRM + email; contact list sync | 300/day | `@getbrevo/brevo` |
| **Resend** | Dev-friendly; React Email templates | 100/day | `resend` |
| **SendGrid** | Enterprise scale; high deliverability | 100/day | `@sendgrid/mail` |
| **Mailgun** | Pay-as-you-go; EU data residency | Trial only | `mailgun.js` |
| **Postmark** | Best deliverability; transactional focus | Trial only | `postmark` |
| **None** | Webhook / custom handler | — | — |

### Every API route does three things
1. **Validate** required fields server-side → `400` + clear error if missing
2. **Notify team** — internal email; full submission table; `replyTo` = submitter's email
3. **Confirm submitter** — auto-reply with CTA-matching copy

**Brevo only — also do:**
4. **Sync contact** — `createContact` with `updateEnabled: true` → upsert into CRM list

**Forms with file upload fields** must submit as `multipart/form-data`, not JSON.
The API route reads the file with `await req.formData()` and attaches it to the notification email.
See `references/email-services.md → File Attachments` for per-service attachment syntax and size limits.

### Environment variables (always list after generating)

| Variable | Description |
|----------|-------------|
| `[SERVICE]_API_KEY` | e.g. `BREVO_API_KEY`, `RESEND_API_KEY`, `SENDGRID_API_KEY` |
| `SENDER_NAME` | Display name for outgoing emails |
| `SENDER_EMAIL` | Verified sender address (must match verified domain) |
| `NOTIFY_EMAIL` | Team inbox for submission notifications |
| `BREVO_LIST_ID` | Brevo only — list ID to add contacts to |

---

## Phase 6 — Post-Generation Checklist

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
   # BREVO_LIST_ID=3   ← Brevo only

🌐 Email service setup:
   1. Create account at [service URL]
   2. Get API key → Settings → API Keys
   3. Verify sender domain (add DNS records they provide)
   4. [Brevo] Create contact list → note the list ID

🧪 Test locally:
   npm run dev → fill form → check team inbox + submitter inbox
   npm run build → confirm 0 TypeScript/build errors
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

## Reference Files

| File | When to read |
|------|--------------|
| `references/form-presets.md` | Phase 3 — always; field definitions per business type |
| `references/email-services.md` | Phase 5 — integration code for all email services; file attachment syntax |
| `references/accessibility-patterns.md` | Pills, star ratings, file upload, conditional/progressive patterns, Typeform-style, multi-step |
