# Changelog

All notable changes to this skill are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [1.6.0] — 2026-05-12

### Changed — `SKILL.md` rewrite (editorial; no behaviour change)

Trimmed `SKILL.md` from **609 → 359 lines** by removing duplication and
moving inline code snippets into the reference files. The skill's flows,
outputs, and triggers are unchanged.

- **Phases renumbered to a flat `1–11`** (was `0 / 1 / 2 / 2.5 / 3 / 4 / 5 / 6 / 7 / 8 / A`). The `0`, `2.5`, and `A` outliers are gone; Audit is now Phase 11.
- **Frontmatter description** tightened from ~18 lines to ~10 while preserving every trigger phrase.
- **Pill-toggle React snippet** moved out of `SKILL.md` into `references/accessibility-patterns.md → Pill toggles`.
- **Component shell** (the 50-line React starter) moved into `references/accessibility-patterns.md → Component shell`.
- **Broken UI-pattern table** (a stray row with no header, left from a prior edit) repaired.
- **Reference table** at the bottom of `SKILL.md` collapsed from 8 rows of prose into a 2-column `file | when to read`.
- **Local-verification output block** kept (it's the template Claude prints), but per-line commentary like `← Brevo only` collapsed into one *"omit lines that don't apply"* instruction above it.

### Added — Two new reference files (single sources of truth)

- **`references/api-contract.md`** — the nine-step API contract (origin → size → sanitize → validate → consents → bot guards → deliver → CRM upsert → log), the delivery quick-pick table, and the full env-var table. Phase 7 (build) and Phase 11 (audit) both point at this file, eliminating the duplication that previously had the nine steps written out twice.
- **`references/audit-rubric.md`** — the Phase 11 category × rubric-source × spot-checks table plus the P0–P3 severity-bucket definitions. Audit findings still trace back to the same rubric files used during generation, but the index now lives in one place.

### Unchanged

`form-presets.md`, `email-services.md`, `field-validation.md`,
`design-context.md`, `security-patterns.md`, `compliance.md`,
`deployment.md`, `.claude-plugin/*`. The skill's behaviour, generated
files, and trigger surface are byte-equivalent to 1.5.0.

---

## [1.5.0] — 2026-05-12

### Added — Design context probe (Phase 2.5)

New `skills/web-forms/references/design-context.md` plus a new **Phase 2.5** in `SKILL.md`. Before any code is generated, the skill runs a bounded, read-only probe of the project to detect:

- A design doc (`design.md` / `DESIGN.md` / `docs/design.md` / `docs/design-system.md`)
- Token sources — `tailwind.config.{js,ts,mjs,cjs}` (`theme.colors`, `borderRadius`, `fontFamily`), then `tokens.json` / `design-tokens.json`, then CSS custom properties in `globals.css`
- Component primitives — shadcn `components/ui/*`, Chakra, MUI, react-aria, or custom `components/Form*.tsx`
- Dark-mode support (`dark:` variants or `[data-theme=dark]`)

When tokens or primitives are detected, the generator substitutes them for inline hex (`bg-primary` instead of `bg-[#HEX]`) and imports existing primitives (`<Button>`, `<Input>`, `<Label>`) instead of hand-rolling markup. Phase 1 Q5 (brand colors) is skipped when tokens already exist. A single `🎨 Design context: …` line is shown to the user before Phase 3 so the probe results are visible and correctable. Phase A audits the existing form against the same probe and flags inline-hex / hand-rolled primitives / missing dark variants as P2 findings.

### Added — Polish Layer (P2 opt-in)

New "Polish Layer (P2 opt-in)" section in `references/accessibility-patterns.md` documenting eight independent upgrades that the generator does **not** include by default but can apply on request or offer during Audit: floating labels, character counter on textareas, button label morph (`Send → Sending → Sent ✓`), error summary at the top (GOV.UK pattern), dark-mode pass, inline help text under labels, optimistic ✓ on valid blur, and smart enter-key behavior for multi-step forms. Phase A.2 audit checklist now cross-links to this section.

### Added — Per-field validation reference

New `skills/web-forms/references/field-validation.md` covering the four field types most often shipped wrong: phone, email, URL/links (LinkedIn, GitHub, portfolio), and date of birth. Same reference drives both Greenfield generation and Audit findings.

- Validation UX principles (validate on blur not on keystroke; specific error copy; pair color with icon + text per WCAG 1.4.1; never disable the submit button; respect `prefers-reduced-motion`).
- Phone: `type=tel` + `inputmode=tel` + `autocomplete=tel`, format-as-you-type with `libphonenumber-js`'s `AsYouType`, blur-time `parsePhoneNumberFromString().isValid()`, server-side E.164 normalization.
- Email: `inputmode=email` + `autocomplete=email` + `autocapitalize=off` + `spellcheck=false`, blur-time trim+lowercase, inline typo suggestion ("Did you mean alice@gmail.com?") against a small popular-domain table, optional disposable-domain reject server-side.
- URL/links: `type=url` + `inputmode=url`, auto-prepend `https://` on blur, `new URL()` validation, server-side protocol allow-list (`http`/`https` only — blocks `javascript:` / `data:`), optional per-field hostname allow-list (e.g. `github.com`).
- Date of birth: native `<input type="date">` with dynamic `min`/`max`, `autocomplete=bday`, UTC `Date.UTC` parse (never `new Date(userString)`), year/month/day age comparison (no leap-year drift), `[minAge, 120]` range. 3-select fallback pattern included.
- Cross-field rules table (phone-or-email, preferred-contact coupling, age-gating).
- Mobile-first throughout: 16px+ font (avoids iOS zoom), 44×44px tap targets.

### Changed

- `SKILL.md` Phase 1: added Q12 (DOB minimum age — 13/16/18/21/none) shown only when the preset includes a DOB field; drives the `max=` attribute and server-side age check.
- `SKILL.md` Phase 4 "Client-side validation": replaced 3-bullet stub with the principles list and a pointer to the new reference.
- `SKILL.md` Phase 5 step 3: widened "email regex" to "format validation per field type" with a pointer to the new reference.
- `SKILL.md` Phase A.2 audit checklist: added a "Field-type validation" row so audits flag missing `type=tel`/`type=url`/`type=date`, missing `inputmode`/`autocomplete`, missing libphonenumber server check, and missing DOB age range as P1 findings.
- `references/security-patterns.md`, `references/accessibility-patterns.md`, `references/form-presets.md`: cross-linked to the new reference at the relevant points; no behavior change.

---

## [1.4.0] — 2026-05-12

### Changed — Distributable as a Claude Code plugin

The repo is now installable via the official plugin marketplace flow, in addition to the existing standalone install:

- Added `.claude-plugin/plugin.json` (plugin manifest) and `.claude-plugin/marketplace.json` (marketplace catalog).
- Moved `SKILL.md` and `references/` into `skills/web-forms/` so the repo matches the plugin layout Claude Code expects.
- Rewrote the README **Installation** section with three copy-paste options: plugin install (`/plugin marketplace add` + `/plugin install`), standalone clone-and-copy (user or project scope), and local `--plugin-dir` testing. Added Requirements and Troubleshooting subsections.

No changes to skill content, phases, or generated code.

---

## [1.3.0] — 2026-05-12

### Added — Audit & Improve mode (existing forms)

The skill now handles two intents through a single entry point:

- **Phase 0 — Intent Triage** (new section at top of `SKILL.md`): routes the request to **Build** (greenfield, Phases 1–8, unchanged) or **Audit & Improve** (existing form, new Phase A). Triage signals: a referenced file path, pasted code, or verbs like *review / audit / edit / improve / check / fix / polish*.
- **Phase A — Audit & Improve** (new section in `SKILL.md`):
  - A.1 locate files (component + API route + env example)
  - A.2 run the checklist audit against the **existing** reference rubrics — security (`security-patterns.md`), a11y (`accessibility-patterns.md`), validation (Phase 4), API contract (Phase 5 9-step list), compliance (`compliance.md`), UI/UX polish (new heuristics section)
  - Findings reported as a Markdown table with **P0 / P1 / P2 / P3 severity** and `file:line` evidence
  - A.3 gated confirmation — user picks fix scope; no edits without approval
  - A.4 apply fixes that reuse the generator's snippets verbatim, smallest diff possible
  - A.5 fallback to greenfield rewrite when the existing form is too far gone to patch
- **`references/accessibility-patterns.md` → UI/UX heuristics for form polish** (new section): visual hierarchy, spacing rhythm, error UX, mobile (`inputmode`/`autocomplete`/tap targets), brand application, microcopy, motion (`prefers-reduced-motion`). Drives the P2 bucket in audits and the final pass in generation.
- **Trigger expansion** in `SKILL.md` frontmatter `description` and `README.md` Triggers list: phrases like *"review my form", "improve this form", "audit my contact form", "edit existing form", "is my form accessible?", "fix form UX"*.

### Changed
- `README.md` now leads with a **Two modes** table (Build vs. Audit & Improve) so the dual purpose is visible before the feature list.
- Reference Files table in `SKILL.md` notes that `accessibility-patterns.md` also powers Phase A's UI/UX rubric.

### Design notes
- **No new rubric file.** The audit reuses every existing reference file as its source of truth, so the generator and the audit stay in lockstep as standards evolve. Only genuinely new content is the UI/UX polish heuristics, which both modes use.
- **Same gating model as Deploy** (Phase 8): the audit is free, edits require explicit user confirmation.

---

## [1.2.0] — 2026-05-12

### Added

#### references/email-services.md — no-3rd-party delivery paths
- **Option 1 — SMTP via Nodemailer**: pooled transporter singleton, full Next.js API route, env vars for Google Workspace / Microsoft 365 / Amazon SES SMTP, deliverability notes
- **Option 2 — Database persistence**: shared `submissions` schema (with `ip_hash` salted-SHA256), three recipes (Prisma, Supabase, raw `pg`), connection-pool guidance for serverless
- **Option 3 — Slack / Discord / Microsoft Teams webhooks**: Block Kit, Embed, and Adaptive Card payloads; retry-with-`Retry-After` helper
- **Option 4** renamed from "No Email / Webhook Only" to "Generic outbound webhook" with clearer framing
- Replaced the prior 40-line webhook stub (which ended in `TODO: save to DB`) with the four concrete recipes above

#### references/compliance.md (new file)
- Phase 1 Q10 consent interview — privacy default, plus optional marketing / ToS / age / custom
- Phase 4 consent UI patterns (single + multi-checkbox, React/Tailwind + plain HTML)
- Rules: required consents unchecked by default, separate checkbox per consent, `target="_blank" rel="noopener"` on links
- Phase 5 server-side `<key>ConsentedAt` timestamp persistence
- Data retention strategies (forever / N-day purge / archive); Postgres, Supabase scheduled function, Vercel Cron recipes
- IP hashing with salted SHA-256 + `IP_SALT` env var — never store raw IP
- Right-to-be-forgotten request handling

#### references/deployment.md (new file)
- Pre-flight check (build clean, local submission verified, `.env.local` gitignored)
- Platform matrix: Vercel, Netlify, Cloudflare Pages, Railway, Fly, Render, Docker/VPS — runtime model + env-var location
- Serverless vs long-running comparison; **in-memory rate limiter must move to Upstash on serverless** (required, not optional)
- Public vs server-only env-var classification — only `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` is browser-visible
- Sender domain verification: SPF + DKIM + DMARC records, `dig` and MXToolbox checks before first send
- Origin allow-list (`ALLOWED_ORIGINS`) tied to API-route step 0
- Production smoke test scripts: real submission, bot-path silent-200, rate-limit fires
- Lighthouse + axe accessibility pass on deployed URL
- Monitoring per platform; Sentry/Axiom/Logflare hookup
- Post-deploy hygiene: `.env.local` audit, key rotation, recurring DMARC review

#### SKILL.md
- Phase 1: Q6 expanded from "Which email service?" to "How should submissions be delivered?" — managed email options plus SMTP/Nodemailer, DB, Slack/Discord/Teams, generic webhook (combinable)
- Phase 1: Q10 added — consent picker (privacy default, optional marketing/ToS/age/custom) with policy-URL capture and placeholder-page scaffolding
- Phase 1: Q11 added — inline success vs thank-you page redirect (for analytics conversion tracking)
- Phase 4: new "Consent layer" subsection — one checkbox per consent, unchecked by default, policy links with `rel="noopener"`
- Phase 5: API route contract expanded from 7 steps to 9 — added **step 0 Origin check** (closes the `ALLOWED_ORIGINS` gap), step 4 consent validation, step 6 multi-channel delivery, step 8 real-failure logging
- Phase 5: Delivery quick-pick table replaces service quick-pick — covers managed + no-3rd-party paths
- Phase 5: Env-vars table expanded with SMTP, DB, chat-webhook, generic-webhook, and `IP_SALT` rows; columns now indicate which delivery option each variable applies to
- Phase 6: renamed to "Post-Generation & **Local Verification**"; deployment steps explicitly excluded; ends with "Ready to deploy?" prompt that defers prod work to Phase 8
- Phase 6 checklist: added consent + privacy status block; added bot-guard sanity-check instruction
- **Phase 8 added — Deploy (user-initiated only)**: strict activation rule (local tests passed + explicit deploy request); 9-step walkthrough referencing `deployment.md`
- Reference Files table: added `compliance.md` and `deployment.md` entries; updated `email-services.md` and `security-patterns.md` descriptions

#### README.md
- Restructured "What it does" — submission delivery now lists managed + no-3rd-party paths explicitly
- Added compliance + Deploy phase bullets
- Structure tree updated with `compliance.md` and `deployment.md`
- Phases table updated: 11 interview questions, 8 phases, Phase 6 is local-only, Phase 8 is the new user-initiated Deploy

### Fixed
- `ALLOWED_ORIGINS` env var was previously listed in Phase 5 but never actually checked by the API-route contract — now enforced as step 0

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
