# web-forms skill

A Claude skill for building production-ready web forms — contact forms, lead capture,
job applications, surveys, requirements gathering, and more.

## What it does

- **Structured interview** before generating any code
- **Field presets** per business type (SaaS, agency, consulting, job application, survey, requirements)
- **UI patterns**: pill toggles, conditional/progressive logic, Typeform-style conversational forms
- **Email integration**: Brevo, Resend, SendGrid, Mailgun, Postmark — including file attachments
- **Accessibility**: WCAG 2.1 AA compliant patterns throughout
- **Multi-step forms** with adaptive step routing based on earlier answers

## Triggers

Use this skill when asked to:
- "Create a form", "build a contact page", "add a lead capture", "make a survey"
- "Set up a job application form", "collect requirements", "add an inquiry form"
- "Integrate Brevo / Resend / SendGrid", "send form emails", "form submission webhook"
- Build a "Typeform-style", "conversational", or "one question at a time" form

## Structure

```
web-forms/
├── SKILL.md                          # Main skill instructions (7 phases)
└── references/
    ├── form-presets.md               # Field definitions per business type
    ├── email-services.md             # Integration code + file attachment support
    └── accessibility-patterns.md    # WCAG patterns, conditional logic, Typeform-style
```

## Phases

| Phase | Description |
|-------|-------------|
| 1 | Interview — 8 questions to shape the form |
| 2 | Identify stack & output format |
| 3 | Select field preset + UI patterns (pills, conditional, Typeform-style) |
| 4 | Build the form component |
| 5 | API route & email integration (with file attachment support) |
| 6 | Post-generation checklist |
| 7 | Multi-step forms (optional) |

## Changelog

See [CHANGELOG.md](./CHANGELOG.md).

## Installation

Install this skill by dropping the `web-forms/` directory into your skills folder,
or by using the `.skill` package file if your environment supports it.
