# web-forms skill

A Claude skill for building production-ready web forms — contact forms, lead capture,
job applications, surveys, requirements gathering, and more.

## Two modes

| Mode | When | What it does |
|------|------|--------------|
| **Build** (greenfield) | "Create a contact form for my SaaS" | Runs the 11-question interview, generates the component + API route, lists env vars, hands off a local-test checklist. |
| **Audit & Improve** (existing form) | "Review my form at `src/app/contact/page.tsx`" / "Improve the UX of this form" / "Is my form accessible?" | Reads your existing files, scores them against the same security / a11y / UX rubrics the generator enforces, returns a prioritized report (P0–P3 with file:line evidence), then applies the fixes you approve. |

Both modes share the same reference files, so a generated form and an audited form converge on the same standards.

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

**Build:**
- "Create a form", "build a contact page", "add a lead capture", "make a survey"
- "Set up a job application form", "collect requirements", "add an inquiry form"
- "Integrate Brevo / Resend / SendGrid", "send form emails", "form submission webhook"
- Build a "Typeform-style", "conversational", or "one question at a time" form

**Audit & Improve:**
- "Review my form", "audit my contact form", "edit existing form", "improve this form"
- "Is my form accessible?", "check my form code", "form code review"
- "Make my form look better", "fix the UX on this form", "polish this form"

## Structure

```
web-form-skill/                       # repo root (also the plugin root)
├── .claude-plugin/
│   ├── plugin.json                   # Plugin manifest
│   └── marketplace.json              # Marketplace catalog (1 plugin)
├── skills/
│   └── web-forms/
│       ├── SKILL.md                  # Main skill: Phase 0 triage → Build (1–8) or Audit (A)
│       └── references/
│           ├── form-presets.md       # Field definitions per business type
│           ├── email-services.md     # Managed services + no-3rd-party paths (SMTP, DB, chat webhooks); attachments
│           ├── accessibility-patterns.md  # WCAG patterns, conditional logic, Typeform-style
│           ├── security-patterns.md  # escapeHtml, rate limiter, honeypot, reCAPTCHA v3, origin guard
│           ├── compliance.md         # Consent checkboxes, policy-link scaffolding, data retention, IP hashing, RTBF
│           └── deployment.md         # User-initiated Deploy phase: platform env vars, DNS verification, smoke tests
├── README.md
└── CHANGELOG.md
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

### Option 1 — Install as a plugin (recommended)

Inside Claude Code, paste these two commands:

```text
/plugin marketplace add venkatmuhil/web-form-skill
/plugin install web-forms@web-forms-marketplace
```

That's it. The skill auto-triggers when you ask Claude to "create a contact form", "audit my form", etc.

- **Update:** `/plugin marketplace update web-forms-marketplace`
- **Uninstall:** `/plugin uninstall web-forms@web-forms-marketplace`

> Plugin skills are namespaced — if needed, invoke explicitly with `/web-forms:<action>`.

### Option 2 — Install as a standalone skill (no plugin system)

Pick a scope:

**User scope** — available in every project on your machine:

```bash
git clone https://github.com/venkatmuhil/web-form-skill.git /tmp/web-form-skill && \
  mkdir -p ~/.claude/skills && \
  cp -R /tmp/web-form-skill/skills/web-forms ~/.claude/skills/ && \
  rm -rf /tmp/web-form-skill
```

**Project scope** — committed with your repo, shared with teammates via git:

```bash
git clone https://github.com/venkatmuhil/web-form-skill.git /tmp/web-form-skill && \
  mkdir -p .claude/skills && \
  cp -R /tmp/web-form-skill/skills/web-forms .claude/skills/ && \
  rm -rf /tmp/web-form-skill
```

Verify with `ls ~/.claude/skills/web-forms/SKILL.md` (or `.claude/skills/web-forms/SKILL.md` for project scope). Restart Claude Code, then ask "create a contact form" — the skill should trigger automatically.

- **Update:** re-run the install one-liner, or `cd ~/.claude/skills/web-forms && git pull` if you installed via clone instead of copy.
- **Uninstall:** `rm -rf ~/.claude/skills/web-forms`

### Option 3 — Test locally without installing

```bash
git clone https://github.com/venkatmuhil/web-form-skill.git
claude --plugin-dir ./web-form-skill
```

### Requirements

- Claude Code CLI installed and authenticated — see the [Claude Code quickstart](https://code.claude.com/docs/en/quickstart).
- Generated form code targets Next.js App Router by default; the skill detects other stacks (Remix, SvelteKit, Astro, plain HTML) during Phase 1's interview.

### Troubleshooting

- **Skill not triggering after install:** restart your Claude Code session, then run `/plugin` (or `/skills`) and confirm `web-forms` is listed.
- **Standalone install — directory name must be exactly `web-forms`** (it must match the `name:` field in `SKILL.md` frontmatter). If you cloned the repo directly into `~/.claude/skills/`, rename the resulting folder to `web-forms`.
- **Plugin not found:** run `/plugin marketplace update web-forms-marketplace` to refresh the catalog, then retry `/plugin install`.
