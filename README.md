# web-forms skill

A Claude skill for building production-ready web forms — contact forms, lead capture,
job applications, surveys, requirements gathering, and more.

## What it does

- **Structured interview** before generating any code
- **Field presets** per business type (SaaS, agency, consulting, job application, survey, requirements)
- **UI patterns**: pill toggles, conditional/progressive logic, Typeform-style conversational forms
- **Submission delivery — with or without 3rd-party services**:
  - Managed email: Brevo, Resend, SendGrid, Mailgun, Postmark (file attachments supported)
  - Self-hosted SMTP via Nodemailer (Google Workspace, M365, SES SMTP, any relay)
  - Database persistence: Postgres, Supabase, Prisma, raw `pg`
  - Slack / Discord / Microsoft Teams webhooks
  - Generic outbound webhook (Zapier / Make / n8n / own backend)
  - Combinable — e.g. DB + Slack with no email at all
- **Accessibility**: WCAG 2.1 AA compliant patterns throughout
- **Multi-step forms** with adaptive step routing based on earlier answers
- **Security by default**: every generated form is production-hardened out of the box
- **Compliance**: consent checkboxes (privacy, marketing, ToS, age, custom) with server-side timestamping, policy-link scaffolding, data retention guidance
- **Separate user-initiated Deploy phase**: platform env-var setup, SPF/DKIM/DMARC verification, production smoke tests, monitoring — runs only when you ask, never as part of generation

## Security defaults (included in every generated form)

No extra setup required — these are baked into the generated code:

| Protection | What it does |
|-----------|--------------|
| `escapeHtml` | Prevents XSS via unsanitized values in HTML email bodies |
| `sanitizeInput` | Strips CRLF (email header injection), trims + caps field lengths |
| Email regex validation | Server-side format check beyond just `if (!email)` |
| Body size guard | Rejects requests > 50 KB before JSON.parse |
| Honeypot field | Hidden input bots auto-fill; real users never touch it |
| Submission time guard | Rejects submissions arriving in < 3 seconds (bot speed) |
| Rate limiter | Max 5 requests / 60s per IP (in-memory; Upstash upgrade path included) |
| Duplicate detection | Same email blocked for 5 minutes after first submission |
| Spam keyword filter | Configurable server-side keyword list checked against message field |

**Optional (Q9 in interview):**
- **reCAPTCHA v3** — invisible CAPTCHA, no checkbox friction, Google score ≥ 0.5 required

## Triggers

Use this skill when asked to:
- "Create a form", "build a contact page", "add a lead capture", "make a survey"
- "Set up a job application form", "collect requirements", "add an inquiry form"
- "Integrate Brevo / Resend / SendGrid", "send form emails", "form submission webhook"
- Build a "Typeform-style", "conversational", or "one question at a time" form

## Structure

```
web-forms/
├── SKILL.md                          # Main skill instructions (8 phases)
└── references/
    ├── form-presets.md               # Field definitions per business type
    ├── email-services.md             # Managed services + no-3rd-party paths (SMTP, DB, chat webhooks); attachments
    ├── accessibility-patterns.md     # WCAG patterns, conditional logic, Typeform-style
    ├── security-patterns.md          # Security utilities: escapeHtml, rate limiter, honeypot, reCAPTCHA v3, origin guard
    ├── compliance.md                 # Consent checkboxes, policy-link scaffolding, data retention, IP hashing, RTBF
    └── deployment.md                 # User-initiated Deploy phase: platform env vars, DNS verification, smoke tests
```

## Phases

| Phase | Description |
|-------|-------------|
| 1 | Interview — 11 questions to shape the form (delivery method, reCAPTCHA, consents, success UX) |
| 2 | Identify stack & output format |
| 3 | Select field preset + UI patterns (pills, conditional, Typeform-style) |
| 4 | Build the form component (consent layer + mandatory security layer) |
| 5 | API route & submission delivery — 8-step ordered contract (origin → size → sanitize → validate → consent → bot guards → deliver → log) |
| 6 | Post-generation & **local verification only** — no deployment steps |
| 7 | Multi-step forms (optional) |
| 8 | **Deploy — user-initiated only.** Walks platform env vars, SPF/DKIM/DMARC verification, production smoke test, Lighthouse/axe a11y pass, monitoring. Runs strictly after local tests pass and the user says "deploy". |

## Changelog

See [CHANGELOG.md](./CHANGELOG.md).

## Installation

Install this skill by dropping the `web-forms/` directory into your skills folder,
or by using the `.skill` package file if your environment supports it.
