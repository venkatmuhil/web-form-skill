# Changelog

All notable changes to this skill are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

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
