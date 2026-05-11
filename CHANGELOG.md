# Changelog

All notable changes to this skill are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [1.1.0] — 2026-05-12

### Added

#### references/security-patterns.md (new file)
- `escapeHtml` utility — prevents XSS via unsanitized values in HTML email bodies
- `sanitizeInput` utility — strips CRLF (email header injection prevention), trims, enforces max lengths per field type
- Server-side email regex validation pattern
- Body size guard (50 KB limit before JSON.parse)
- CORS origin check via `ALLOWED_ORIGINS` env var
- In-memory rate limiter (`utils/rateLimiter.ts`) — 5 req / 60s per IP with auto cleanup
- Upstash Redis rate limiter — documented upgrade path for multi-instance / serverless deployments
- Honeypot field — React/TSX + plain HTML versions with silent `200` server check
- Submission time guard — client timestamp + server-side 3-second minimum check
- Duplicate submission detection (`utils/duplicateGuard.ts`) — same email blocked for 5-minute TTL window
- Spam keyword filter (`utils/spamFilter.ts`) — configurable server-side keyword list
- reCAPTCHA v3 integration — invisible CAPTCHA; client `GoogleReCaptchaProvider` + `executeRecaptcha`; server `verifyRecaptcha` with score ≥ 0.5 threshold
- Security checklist block for Phase 6 output

#### SKILL.md
- Phase 1: Q9 added — reCAPTCHA v3 opt-in (default Yes for public forms)
- Phase 4: "Security layer (non-negotiable)" subsection — honeypot, submission timer, reCAPTCHA
- Phase 5: "Every API route does three things" replaced with 7-step ordered security-first contract
- Phase 5: `ALLOWED_ORIGINS`, `NEXT_PUBLIC_RECAPTCHA_SITE_KEY`, `RECAPTCHA_SECRET_KEY` added to env vars table
- Phase 6: `🔒 Security defaults included` checklist block added to post-generation output
- Reference Files table: `references/security-patterns.md` entry added

#### references/email-services.md
- All 5 service templates (Brevo, Resend, SendGrid, Mailgun, Postmark) and webhook stub updated:
  - Body size guard before JSON.parse
  - Full input sanitization block (`sanitizeInput` on all fields)
  - Email regex validation
  - Complete bot-guard chain: rate limit → honeypot → time guard → duplicate → spam filter
  - Commented reCAPTCHA v3 hook (enabled when Q9 = Yes)
  - `escapeHtml()` applied to every `${value}` interpolated into HTML email bodies
  - Mailgun `h:Reply-To` header injection specifically hardened

#### references/form-presets.md
- New **Security Fields** section (mandatory, before Common Field Patterns) — honeypot + form timer with HTML and React patterns
- Honeypot entry in Common Field Patterns replaced with cross-reference to security-patterns.md

### Changed
- README updated to document security defaults table and updated structure/phases

---

## [1.0.0] — 2026-05-11

### Added

#### Core skill (SKILL.md)
- Phase 1: Structured 8-question interview before any code is generated
- Phase 2: Stack and output format detection (Next.js App/Pages Router, plain React, HTML, Vue/Svelte)
- Phase 3: Business-type → field preset mapping (consulting, SaaS, agency, job-application, survey, requirements, contact)
- Phase 3: UI pattern rules — pill toggles, single-select pill rows, file inputs
- Phase 3: **Conditional / Progressive Logic** — field show/hide, section skip, adaptive multi-step (optional layer for all presets)
- Phase 3: **Conversational / Typeform-style Mode** — one question at a time, full-screen slides, keyboard navigation, progress bar + counter, animated transitions
- Phase 4: React component shell with loading, error, and submitted states
- Phase 4: WCAG 2.1 AA accessibility rules (labels, required markers, error roles, focus management, aria-busy)
- Phase 4: Client-side validation (HTML5 + JS live blur feedback)
- Phase 5: API route pattern (validate → notify team → confirm submitter)
- Phase 5: **File upload handling** — `multipart/form-data` submission, `req.formData()` server parsing, per-service attachment syntax
- Phase 6: Post-generation checklist (files, install, env vars, email setup, local test)
- Phase 7: Multi-step form guidance with adaptive routing

#### references/form-presets.md
- Field definitions for all 7 presets: consulting, saas, agency, job-application, survey, requirements, contact

#### references/email-services.md
- Full integration code for: Brevo, Resend, SendGrid, Mailgun, Postmark, webhook-only
- UTM tracking pattern (all services)
- **File Attachments** section: `FormData` frontend pattern, `req.formData()` server pattern, per-service attachment syntax, size limits table

#### references/accessibility-patterns.md
- Core WCAG rules: labels, required fields, error messages, focus management, submit button states
- Complex field patterns: star rating, NPS scale, file upload with validation, checkbox group
- **Conditional & Progressive Patterns**: Pattern 1 (field show/hide), Pattern 2 (section skip), Pattern 3 (adaptive multi-step) — all with HTML and React/Tailwind versions
- **Conversational / Typeform-style Pattern**: question definition schema, state & navigation, keyboard handler, CSS `@keyframes` + Framer Motion variants, progress bar, `QuestionInput` renderer, full layout shell, accessibility notes
- Multi-step form pattern (fixed steps)
- WCAG quick reference table
