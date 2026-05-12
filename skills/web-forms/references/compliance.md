# Compliance — Consent, Privacy Links, Data Retention

Read this file in **Phase 1** (interview — ask which consents are needed) and **Phase 4** (render the consent UI). Also referenced from Phase 5 (server-side consent timestamping) and the Deploy phase (data retention job).

The skill is not a substitute for legal advice. It gives the user a reasonable, GDPR-aligned default and lets them override.

---

## Phase 1 — Consent interview

Add this question after the existing Q9:

> **Q10. Which consents do you need on the form?**
>
> Default: **Privacy / GDPR consent** (always recommended if the form collects name + email).
>
> Optional additions — pick any that apply:
> - **Marketing opt-in** — separate checkbox if the user is being added to a mailing list
> - **Terms of Service** acceptance — when the form is also signup-adjacent
> - **Age confirmation** — "I am 18+" / "I am 16+" (required for some jurisdictions)
> - **Cookie / tracking** consent — only if the form runs analytics that need explicit consent
> - **Custom** — any other consent the user names

For each consent the user picks, also ask:
> Do you have the linked page ready? Paste the URL — or tell me if you need me to scaffold a placeholder route.

If the user does **not** have a privacy policy URL: offer to scaffold `/privacy` and `/terms` placeholder pages with a clearly marked "FILL ME IN" body. Better to ship a stub than to ship a checkbox that links to `#`.

---

## Phase 4 — Consent UI

### Single consent (most common — privacy only)

React/Tailwind:
```tsx
<div className="flex items-start gap-2">
  <input
    id="privacyConsent"
    name="privacyConsent"
    type="checkbox"
    required
    checked={form.privacyConsent}
    onChange={e => setForm(f => ({ ...f, privacyConsent: e.target.checked }))}
    className="mt-1"
  />
  <label htmlFor="privacyConsent" className="text-sm text-gray-700">
    I have read and agree to the{" "}
    <a href="/privacy" target="_blank" rel="noopener" className="underline">
      privacy policy
    </a>. *
  </label>
</div>
```

Plain HTML:
```html
<div class="consent">
  <input id="privacyConsent" name="privacyConsent" type="checkbox" required>
  <label for="privacyConsent">
    I have read and agree to the
    <a href="/privacy" target="_blank" rel="noopener">privacy policy</a>. *
  </label>
</div>
```

### Multiple consents — checkbox group

```tsx
const CONSENTS = [
  { key: "privacy",   required: true,  label: "I agree to the",                href: "/privacy",   text: "privacy policy" },
  { key: "terms",     required: true,  label: "I accept the",                  href: "/terms",     text: "terms of service" },
  { key: "marketing", required: false, label: "I want to receive updates per the", href: "/privacy", text: "privacy policy" },
];

<fieldset className="space-y-2 text-sm">
  <legend className="sr-only">Consents</legend>
  {CONSENTS.map(c => (
    <div key={c.key} className="flex items-start gap-2">
      <input
        id={`${c.key}Consent`}
        type="checkbox"
        required={c.required}
        checked={form[`${c.key}Consent`]}
        onChange={e => setForm(f => ({ ...f, [`${c.key}Consent`]: e.target.checked }))}
        className="mt-1"
      />
      <label htmlFor={`${c.key}Consent`} className="text-gray-700">
        {c.label}{" "}
        <a href={c.href} target="_blank" rel="noopener" className="underline">{c.text}</a>
        {c.required && " *"}
      </label>
    </div>
  ))}
</fieldset>
```

### Rules

- **Required consents** use the `required` attribute — HTML5 will block submit until checked. Visual `*` matches the page legend "* Required".
- **Required consents must NOT be pre-checked**. Pre-ticked boxes are non-compliant under GDPR (Planet49 ruling). Default `checked` = `false`.
- Optional consents (marketing) may be unchecked by default; pre-ticking them is also non-compliant.
- Use **separate** checkboxes per consent. One checkbox for "privacy + marketing" is not valid consent.
- `target="_blank" rel="noopener"` on every policy link.

---

## Phase 5 — Server-side timestamping

For every consent the form collects, store a timestamp alongside the submission. A boolean alone is not enough — regulators ask "when did the user consent?" and you need an answer.

```ts
const now = new Date().toISOString();

const consentTimestamps: Record<string, string | null> = {
  privacyConsentedAt:   body.privacyConsent   ? now : null,
  termsConsentedAt:     body.termsConsent     ? now : null,
  marketingConsentedAt: body.marketingConsent ? now : null,
};

// Reject the request if a required consent is missing
if (!consentTimestamps.privacyConsentedAt) {
  return NextResponse.json({ error: "Privacy consent required." }, { status: 400 });
}
```

Persist these alongside the submission:
- DB path → add `privacy_consented_at timestamptz`, `marketing_consented_at timestamptz` columns
- Email path → include them in the notification email body so the team has the record
- Webhook path → include them in the forwarded payload

Withdrawal of consent is a separate UX (usually an unsubscribe link or `/account/privacy` page) — out of scope for the form skill, but flag it to the user as a downstream concern.

---

## Data retention

Personal data collected via a form must have a retention policy. Pick one and document it:

| Strategy | When it fits | How |
|----------|--------------|-----|
| **Forever** (until manual delete) | Low volume, B2B leads owned by sales | No automation; user-driven delete |
| **N-day rolling purge** | High-volume contact / survey forms | Scheduled job deletes rows where `created_at < now() - interval 'N days'` |
| **N days then archive** | Compliance-sensitive (HR, healthcare) | Move to cold storage, drop PII columns, keep aggregate stats |

Default suggestion: **90 days** for general contact forms unless the user has a CRM that takes ownership of the record. Marketing-opt-in records typically need longer (until unsubscribe).

### Retention purge — recipes

**Postgres scheduled job (cron extension or external scheduler):**
```sql
DELETE FROM submissions
WHERE created_at < NOW() - INTERVAL '90 days';
```

**Supabase scheduled function (`supabase/functions/purge-submissions/index.ts`):**
```ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

Deno.serve(async () => {
  const supabase = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
  const cutoff = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000).toISOString();
  const { error, count } = await supabase
    .from("submissions").delete({ count: "exact" }).lt("created_at", cutoff);
  return new Response(JSON.stringify({ deleted: count, error }), {
    headers: { "Content-Type": "application/json" },
  });
});
```
Schedule it via Supabase Cron (Database → Cron Jobs → daily at 03:00).

**Vercel Cron** — add to `vercel.json`:
```json
{
  "crons": [
    { "path": "/api/cron/purge-submissions", "schedule": "0 3 * * *" }
  ]
}
```
Then a route at `src/app/api/cron/purge-submissions/route.ts` that runs the DELETE. Guard it with `Authorization: Bearer ${CRON_SECRET}` so it can't be triggered externally.

---

## IP address handling

IP addresses are PII under GDPR. The skill's rate limiter and duplicate guard need an IP-derived key, but you do **not** need to store the raw IP.

The recommended pattern:
```ts
import { createHash } from "crypto";
const ipHash = createHash("sha256").update(ip + process.env.IP_SALT).digest("hex");
```

Store `ipHash`, never raw `ip`. The salt (`IP_SALT`) is a server-only secret — rotate it yearly to make historical rows un-correlatable.

For the in-memory rate limiter, using the raw IP for the in-process key is fine — it never persists. The hashing only matters for what you save.

---

## Right-to-be-forgotten requests

If a submitter emails asking for deletion:
1. Look up by their email address (`SELECT id FROM submissions WHERE email = $1`).
2. Delete those rows.
3. Email confirmation back.

If you have a CRM (Brevo contact list), delete from there too — the `BREVO_LIST_ID` contact upsert needs a manual delete step that the form doesn't trigger.

The skill does not generate a deletion endpoint by default — that's a separate request. But flag this gap to the user during the consent question if they're collecting GDPR-protected data.

---

## Quick checklist (output during Phase 6)

```
🛡️  Consent + privacy:
   ✅ Required consents are unchecked by default
   ✅ Each consent has its own checkbox + dedicated policy link
   ✅ Server stores <consent>ConsentedAt timestamps
   [ ] Privacy policy page exists and is filled in: /privacy
   [ ] Terms of service page exists and is filled in: /terms       (if applicable)
   [ ] Data retention policy decided (default: 90-day purge)
   [ ] Deletion request process documented
```
