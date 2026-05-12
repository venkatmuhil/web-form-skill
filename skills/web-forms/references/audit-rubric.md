# Audit Rubric (Phase 11)

The rubric the audit phase walks for an existing form. Every category points
at the same reference file used during generation — standards stay in lockstep
across both modes.

Mark each row **✅ pass** / **⚠️ partial** / **❌ missing** with `file:line`
evidence in the audit report.

---

## Categories

| Category | Rubric source | Required spot-checks |
|----------|---------------|----------------------|
| Security | [`security-patterns.md`](security-patterns.md) | `escapeHtml` on all `${value}` in email HTML; `sanitizeInput` on text fields; body-size guard (50 KB); honeypot; submission-time guard (3 s); rate limiter; duplicate-email guard; spam filter; `ALLOWED_ORIGINS` step-0 check; reCAPTCHA v3 if public |
| Accessibility | [`accessibility-patterns.md`](accessibility-patterns.md) | every input has `<label for>` or `aria-label`; radio / checkbox groups wrapped in `<fieldset><legend>`; errors use `role="alert"` or `aria-live`; errors not color-only; `required` + visual asterisk + page legend; submit `aria-busy` while loading; visible focus ring (no bare `outline:none`); focus moves to first invalid field on submit |
| Validation | `SKILL.md` Phase 6 | HTML5 attrs (`required`, `type=email`, `minlength`, `pattern`); server-side regex + length caps; submit disabled only while loading; cross-field rules where relevant |
| Field-type validation | [`field-validation.md`](field-validation.md) | phone uses `type=tel` + `inputmode=tel` + `autocomplete=tel` + `parsePhoneNumberFromString().isValid()` server-side (stored E.164); URL fields use `type=url` + `inputmode=url` + `new URL()` server-side with protocol allow-list (`http`/`https`); DOB uses `type=date` + `min`/`max` + `autocomplete=bday` + UTC `Date.UTC` age check; email has on-blur trim+lowercase, typo-suggestion ("Did you mean…"), `autocapitalize=off` + `spellcheck=false`. Validate on blur; never disable submit; pair color with icon + text. |
| Delivery & API contract | [`api-contract.md`](api-contract.md) + [`email-services.md`](email-services.md) | all nine steps present and in order (origin → size → sanitize → validate → consents → bot guards → deliver → CRM upsert → log); bot-guard paths return silent `200`; HTML email bodies escape every interpolated value |
| Compliance | [`compliance.md`](compliance.md) | one checkbox per consent (no bundling); required consents unchecked by default; server records `<key>ConsentedAt` timestamps; policy links `target="_blank" rel="noopener"`; placeholder `/privacy` and `/terms` routes exist if linked |
| UI / UX polish | [`accessibility-patterns.md → UI/UX heuristics`](accessibility-patterns.md) + Polish Layer | visual hierarchy, spacing rhythm, inline error placement, ≥ 44 px tap targets, `inputmode` + `autocomplete`, brand accent on CTA + focus, microcopy, `prefers-reduced-motion`; opt-in extras: floating labels, character counters, button label morph, error-summary anchor list, dark-mode pass |
| Design system fit | [`design-context.md`](design-context.md) | inline hex `bg-[#xxx]` when `tailwind.config` defines a token; hand-rolled `<input>` when `components/ui/input.tsx` exists; missing `dark:` variants when the rest of the project ships dark mode; ignored tokens from `design.md` |

---

## Severity buckets

- **P0** — security or correctness: XSS, missing sanitization, no rate
  limit, missing origin check, broken submission.
- **P1** — WCAG 2.1 AA accessibility violations and required-consent gaps.
- **P2** — UI/UX polish: spacing, hierarchy, tap targets, microcopy.
- **P3** — nice-to-have: motion, advanced patterns, conversational mode.
