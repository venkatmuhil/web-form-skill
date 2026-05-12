# Field Validation — Per-Type UX & Code Reference

Copy-paste-ready validation patterns for the four field types that most often
go wrong: **phone, email, URL (links / GitHub / LinkedIn / portfolio), and
date of birth**. Use this file from Phase 4 (form component) and Phase 5 (API
route) — the same snippets serve both Greenfield generation and Audit fixes.

> Inspired by Timothy Graf, *Mastering Form Design UX: Input Validation Best
> Practices* (LinkedIn). Treat validation as a UX surface, not just a server gate.

---

## Validation UX Principles (apply to every field type below)

1. **Validate on blur, not on every keystroke.** Live-validating mid-typing
   ("Invalid email" while the user is still typing the `@`) is hostile.
   Re-validate on `change` only **after** the field has been blurred once and
   shown an error — at that point, fix-as-you-type is welcome.
2. **Specific, actionable error copy.** "Invalid" tells the user nothing.
   "Email must include @" or "Phone needs a country code, e.g. +1 555 123 4567"
   does. Each section below ships an error-copy table — use it verbatim.
3. **Success state is subtle.** A small ✓ inside the input is enough. Avoid a
   loud green halo on every valid field — it competes with the actual error.
4. **Pair color with icon + text** (WCAG 1.4.1, "Use of Color"). Red border
   alone is invisible to ~8% of male users. Always: `border-red-500` **plus**
   ⚠ icon **plus** the text in `role="alert"`.
5. **Never disable the submit button.** Disabled buttons hide the *reason* the
   form is unsubmittable. Let the user submit, then surface the errors and
   move focus to the first invalid field (already covered by SKILL.md Phase 4).
6. **Respect `prefers-reduced-motion`.** Wrap any shake-on-error or
   slide-in-error animations in a `@media (prefers-reduced-motion: no-preference)`
   guard.
7. **Mobile defaults — non-negotiable for every field below:**
   - Input font-size **≥ 16px** (smaller triggers iOS zoom-on-focus)
   - Tap target **≥ 44 × 44 px** (Apple HIG)
   - Always set `type` + `inputmode` + `autocomplete` — saves users 5–15
     seconds per form and reduces typos massively

---

## 1. Mobile / Phone (`type="tel"`)

### Why `type="tel"` (not `type="number"`)
`type="number"` strips leading zeros, blocks `+`, parses `e` as scientific
notation, and shows a spinner that lets users increment a phone number. Always
use `type="tel"` for phone fields.

### Install (recommended)

```bash
npm install libphonenumber-js
```

`libphonenumber-js` is a tiny port of Google's `libphonenumber` (~145 KB → ~75 KB
gzipped for `min` build). It handles 200+ country formats. Format-as-you-type
+ E.164 normalization in one library.

### Client — React / TSX

```tsx
import { useState } from "react";
import { AsYouType, parsePhoneNumberFromString, type CountryCode } from "libphonenumber-js";

const DEFAULT_COUNTRY: CountryCode = "US"; // change per audience

const [phone,    setPhone]    = useState("");
const [phoneErr, setPhoneErr] = useState<string | null>(null);
const [touched,  setTouched]  = useState(false);

const onPhoneChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  const formatted = new AsYouType(DEFAULT_COUNTRY).input(e.target.value);
  setPhone(formatted);
  if (touched) validatePhone(formatted); // re-validate live only after first blur
};

const validatePhone = (value: string): boolean => {
  if (!value) {
    setPhoneErr("Phone number is required.");
    return false;
  }
  const parsed = parsePhoneNumberFromString(value, DEFAULT_COUNTRY);
  if (!parsed || !parsed.isValid()) {
    setPhoneErr("Enter a valid phone number with country code, e.g. +1 555 123 4567.");
    return false;
  }
  setPhoneErr(null);
  return true;
};

<div>
  <label htmlFor="phone" className="block font-medium mb-1">
    Phone <span aria-hidden="true">*</span>
  </label>
  <input
    id="phone"
    name="phone"
    type="tel"
    inputMode="tel"
    autoComplete="tel"
    required
    aria-required="true"
    aria-invalid={phoneErr ? "true" : undefined}
    aria-describedby={phoneErr ? "phone-error" : undefined}
    value={phone}
    onChange={onPhoneChange}
    onBlur={() => { setTouched(true); validatePhone(phone); }}
    placeholder="+1 555 123 4567"
    className="w-full min-h-[44px] text-base px-3 py-2 border rounded
               focus:outline-none focus-visible:ring-2 focus-visible:ring-[#ACCENT]
               aria-[invalid=true]:border-red-500"
  />
  {phoneErr && (
    <p id="phone-error" role="alert" className="mt-1 text-sm text-red-600 flex items-center gap-1">
      <span aria-hidden="true">⚠</span> {phoneErr}
    </p>
  )}
</div>

// On submit — normalize to E.164 before POSTing
const e164 = parsePhoneNumberFromString(phone, DEFAULT_COUNTRY)?.number ?? phone;
body: JSON.stringify({ ...form, phone: e164, _t: formLoadTime }),
```

### Client — Plain HTML

```html
<label for="phone">Phone <span aria-hidden="true">*</span></label>
<input
  type="tel"
  id="phone"
  name="phone"
  inputmode="tel"
  autocomplete="tel"
  required
  aria-required="true"
  placeholder="+1 555 123 4567"
  pattern="^\+?[0-9\s().-]{7,20}$"
  style="font-size:16px;min-height:44px;width:100%">
<p id="phone-error" role="alert" class="error" hidden></p>
<!-- Pair with libphonenumber-js loaded via <script type="module"> for AsYouType + isValid -->
```

`pattern` here is a permissive client-side hint only — server is authoritative.

### Server — API route

```ts
// route.ts (after sanitizeInput)
import { parsePhoneNumberFromString } from "libphonenumber-js";

const phoneRaw = sanitizeInput(body.phone, 30);
const phoneParsed = phoneRaw ? parsePhoneNumberFromString(phoneRaw, "US") : null;

if (phoneRaw && (!phoneParsed || !phoneParsed.isValid())) {
  return NextResponse.json({ error: "Invalid phone number." }, { status: 400 });
}
const phoneE164 = phoneParsed?.number ?? ""; // store this, not the raw input
```

**Why re-parse on the server:** the client may be bypassed. Trust only the
parsed `.number` (E.164 string like `+15551234567`) — never the raw input.

### Country code — three options

| Option | When to use |
|--------|-------------|
| **Single default country** (shown above) | Audience is single-country (e.g. US-only B2C) |
| **Country `<select>` next to input** | Multi-country audience; pass selected ISO code as the `defaultCountry` arg |
| **Force `+` prefix, no default** | International B2B; users always type their full E.164 |

### Phone error copy

| Situation | Copy |
|-----------|------|
| Empty + required | "Phone number is required." |
| Fails `parsePhoneNumberFromString().isValid()` | "Enter a valid phone number with country code, e.g. +1 555 123 4567." |
| Wrong country (allow-list mode) | "We can only accept US/Canada numbers right now." |
| Too short / too long | (covered by `isValid()` — same message above) |

---

## 2. Email (`type="email"`)

### Client — React / TSX

```tsx
const [email,    setEmail]    = useState("");
const [emailErr, setEmailErr] = useState<string | null>(null);
const [emailHint, setEmailHint] = useState<string | null>(null);
const [touched,  setTouched]  = useState(false);

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;

// Lightweight typo table — extend as needed (no extra package required)
const COMMON_DOMAINS: Record<string, string> = {
  "gmial.com":   "gmail.com",
  "gmal.com":    "gmail.com",
  "gnail.com":   "gmail.com",
  "yhaoo.com":   "yahoo.com",
  "yaho.com":    "yahoo.com",
  "hotnail.com": "hotmail.com",
  "hotmal.com":  "hotmail.com",
  "outlok.com":  "outlook.com",
  "iclould.com": "icloud.com",
};

const suggestEmail = (value: string): string | null => {
  const at = value.lastIndexOf("@");
  if (at < 0) return null;
  const domain = value.slice(at + 1).toLowerCase();
  const fix = COMMON_DOMAINS[domain];
  return fix ? `${value.slice(0, at + 1)}${fix}` : null;
};

const onEmailBlur = () => {
  setTouched(true);
  const cleaned = email.trim().toLowerCase();
  if (cleaned !== email) setEmail(cleaned);

  if (!cleaned)              { setEmailErr("Email is required.");                  setEmailHint(null); return; }
  if (!EMAIL_RE.test(cleaned)) { setEmailErr("Email must look like name@example.com."); setEmailHint(null); return; }
  setEmailErr(null);
  setEmailHint(suggestEmail(cleaned)); // suggestion is informational, not an error
};

<div>
  <label htmlFor="email" className="block font-medium mb-1">
    Email <span aria-hidden="true">*</span>
  </label>
  <input
    id="email"
    name="email"
    type="email"
    inputMode="email"
    autoComplete="email"
    autoCapitalize="off"
    spellCheck={false}
    required
    aria-required="true"
    aria-invalid={emailErr ? "true" : undefined}
    aria-describedby={emailErr ? "email-error" : emailHint ? "email-hint" : undefined}
    value={email}
    onChange={e => { setEmail(e.target.value); if (touched) setEmailErr(null); }}
    onBlur={onEmailBlur}
    placeholder="you@company.com"
    className="w-full min-h-[44px] text-base px-3 py-2 border rounded
               focus:outline-none focus-visible:ring-2 focus-visible:ring-[#ACCENT]
               aria-[invalid=true]:border-red-500"
  />
  {emailErr && (
    <p id="email-error" role="alert" className="mt-1 text-sm text-red-600 flex items-center gap-1">
      <span aria-hidden="true">⚠</span> {emailErr}
    </p>
  )}
  {!emailErr && emailHint && (
    <p id="email-hint" className="mt-1 text-sm text-gray-700">
      Did you mean{" "}
      <button
        type="button"
        onClick={() => { setEmail(emailHint); setEmailHint(null); }}
        className="underline text-[#ACCENT]"
      >
        {emailHint}
      </button>
      ?
    </p>
  )}
</div>
```

### Client — Plain HTML

```html
<label for="email">Email <span aria-hidden="true">*</span></label>
<input
  type="email"
  id="email"
  name="email"
  inputmode="email"
  autocomplete="email"
  autocapitalize="off"
  spellcheck="false"
  required
  aria-required="true"
  placeholder="you@company.com"
  style="font-size:16px;min-height:44px;width:100%">
<p id="email-error" role="alert" class="error" hidden></p>
<p id="email-hint" class="hint" hidden></p>
```

### Server

The existing `EMAIL_RE` check in `security-patterns.md → Standard Validation Block`
remains authoritative — keep it. Optional additions:

```ts
// Optional: reject disposable domains (opt-in for high-value forms only)
// List source: https://github.com/disposable-email-domains/disposable-email-domains
import disposable from "./disposable-domains.json"; // refresh quarterly
const domain = email.split("@")[1]?.toLowerCase();
if (domain && (disposable as string[]).includes(domain)) {
  return NextResponse.json({ error: "Please use a non-disposable email address." }, { status: 400 });
}
```

Skip the disposable check for surveys, NPS, or feedback forms — false-positive
cost is too high. Enable it for paid trials, free credits, or anything with
abuse risk.

### Email error copy

| Situation | Copy |
|-----------|------|
| Empty + required | "Email is required." |
| Fails regex | "Email must look like name@example.com." |
| Typo suggestion (informational, not an error) | "Did you mean alice@gmail.com?" (one-click fix) |
| Disposable domain (opt-in) | "Please use a non-disposable email address." |

---

## 3. Links / URL (`type="url"`)

Used for LinkedIn, GitHub, portfolio, personal site, company URL, etc.

### Client — React / TSX

```tsx
const [url,    setUrl]    = useState("");
const [urlErr, setUrlErr] = useState<string | null>(null);

// Optional per-field hostname allow-list. Empty = accept any http(s) URL.
const ALLOWED_HOSTS = ["github.com", "www.github.com"]; // example: GitHub-only field

const normalizeUrl = (raw: string): string => {
  const trimmed = raw.trim();
  if (!trimmed) return "";
  if (!/^https?:\/\//i.test(trimmed)) return `https://${trimmed}`;
  return trimmed;
};

const validateUrl = (raw: string): boolean => {
  const value = normalizeUrl(raw);
  if (!value) { setUrlErr(null); return true; } // empty = optional
  let parsed: URL;
  try { parsed = new URL(value); }
  catch { setUrlErr("Looks like a typo — try https://github.com/your-handle."); return false; }

  if (!/^https?:$/.test(parsed.protocol)) {
    setUrlErr("Only http and https links are accepted."); return false;
  }
  if (ALLOWED_HOSTS.length && !ALLOWED_HOSTS.includes(parsed.hostname)) {
    setUrlErr(`Please use a ${ALLOWED_HOSTS[0]} link.`); return false;
  }
  setUrlErr(null);
  setUrl(value); // commit the normalized form back to the input
  return true;
};

<div>
  <label htmlFor="github" className="block font-medium mb-1">GitHub profile</label>
  <input
    id="github"
    name="github"
    type="url"
    inputMode="url"
    autoComplete="url"
    autoCapitalize="off"
    spellCheck={false}
    aria-invalid={urlErr ? "true" : undefined}
    aria-describedby={urlErr ? "github-error" : undefined}
    value={url}
    onChange={e => { setUrl(e.target.value); setUrlErr(null); }}
    onBlur={() => validateUrl(url)}
    placeholder="https://github.com/your-handle"
    className="w-full min-h-[44px] text-base px-3 py-2 border rounded
               focus:outline-none focus-visible:ring-2 focus-visible:ring-[#ACCENT]
               aria-[invalid=true]:border-red-500"
  />
  {urlErr && (
    <p id="github-error" role="alert" className="mt-1 text-sm text-red-600 flex items-center gap-1">
      <span aria-hidden="true">⚠</span> {urlErr}
    </p>
  )}
</div>
```

### Client — Plain HTML

```html
<label for="github">GitHub profile</label>
<input
  type="url"
  id="github"
  name="github"
  inputmode="url"
  autocomplete="url"
  autocapitalize="off"
  spellcheck="false"
  placeholder="https://github.com/your-handle"
  pattern="^(https?://).+\..+"
  style="font-size:16px;min-height:44px;width:100%">
<p id="github-error" role="alert" class="error" hidden></p>
```

### Server

```ts
const githubRaw = sanitizeInput(body.github, 500);

if (githubRaw) {
  let parsed: URL;
  try { parsed = new URL(githubRaw); }
  catch { return NextResponse.json({ error: "Invalid URL." }, { status: 400 }); }

  // Reject non-http(s) — blocks javascript:, data:, file:, etc.
  // (Defense in depth on top of escapeHtml — see security-patterns.md)
  if (!/^https?:$/.test(parsed.protocol)) {
    return NextResponse.json({ error: "Only http and https links are accepted." }, { status: 400 });
  }

  // Optional per-field hostname allow-list
  const allowed = ["github.com", "www.github.com"];
  if (allowed.length && !allowed.includes(parsed.hostname)) {
    return NextResponse.json({ error: `Please provide a ${allowed[0]} link.` }, { status: 400 });
  }
}
```

**Security note:** `escapeHtml` (already in every email body) prevents XSS at
render time, but the protocol check here prevents `javascript:` URLs from ever
landing in your DB or being clicked from a notification email by a teammate.
See [security-patterns.md](security-patterns.md) for the wider security layer.

### URL error copy

| Situation | Copy |
|-----------|------|
| Fails `new URL()` | "Looks like a typo — try https://github.com/your-handle." |
| Non-`http(s):` scheme | "Only http and https links are accepted." |
| Hostname not in allow-list | "Please use a github.com link." (substitute domain) |

---

## 4. Date of Birth (`type="date"`)

### Default: native `<input type="date">`
Mobile uses the OS picker (iOS wheel, Android calendar) — that's the best UX
you can ship without writing a custom date picker. Desktop browsers render
their own widget. Set `min` and `max` to constrain to the valid age window.

### Client — React / TSX

```tsx
// minAge: 13 (COPPA), 16 (GDPR-K), 18 (alcohol/finance), 21 (cannabis), or 0 (none)
const minAge = 13;
const today  = new Date();
const maxDob = new Date(today.getFullYear() - minAge, today.getMonth(), today.getDate())
                 .toISOString().slice(0, 10);
const minDob = new Date(today.getFullYear() - 120,    today.getMonth(), today.getDate())
                 .toISOString().slice(0, 10);

const [dob,    setDob]    = useState("");
const [dobErr, setDobErr] = useState<string | null>(null);

const validateDob = (value: string): boolean => {
  if (!value) { setDobErr("Date of birth is required."); return false; }
  // Parse explicitly as UTC to avoid timezone drift (don't use new Date(value))
  const m = /^(\d{4})-(\d{2})-(\d{2})$/.exec(value);
  if (!m) { setDobErr("Use the format YYYY-MM-DD."); return false; }
  const [, y, mo, d] = m.map(Number) as [string, number, number, number];
  const dobDate = new Date(Date.UTC(y as any, mo - 1, d));

  if (dobDate.getUTCFullYear() !== Number(y) || dobDate > today) {
    setDobErr("Enter a real date in the past."); return false;
  }
  // Age via year/month/day comparison — no leap-year drift
  let age = today.getUTCFullYear() - Number(y);
  const beforeBirthdayThisYear =
    today.getUTCMonth() < mo - 1 ||
    (today.getUTCMonth() === mo - 1 && today.getUTCDate() < d);
  if (beforeBirthdayThisYear) age--;

  if (age < minAge) { setDobErr(`You must be at least ${minAge} years old.`); return false; }
  if (age > 120)    { setDobErr("Please enter a valid date of birth.");        return false; }
  setDobErr(null);
  return true;
};

<div>
  <label htmlFor="dob" className="block font-medium mb-1">
    Date of birth <span aria-hidden="true">*</span>
  </label>
  <input
    id="dob"
    name="dob"
    type="date"
    autoComplete="bday"
    required
    aria-required="true"
    aria-invalid={dobErr ? "true" : undefined}
    aria-describedby={dobErr ? "dob-error" : "dob-help"}
    min={minDob}
    max={maxDob}
    value={dob}
    onChange={e => { setDob(e.target.value); setDobErr(null); }}
    onBlur={() => validateDob(dob)}
    className="w-full min-h-[44px] text-base px-3 py-2 border rounded
               focus:outline-none focus-visible:ring-2 focus-visible:ring-[#ACCENT]
               aria-[invalid=true]:border-red-500"
  />
  <p id="dob-help" className="mt-1 text-xs text-gray-500">You must be at least {minAge} years old.</p>
  {dobErr && (
    <p id="dob-error" role="alert" className="mt-1 text-sm text-red-600 flex items-center gap-1">
      <span aria-hidden="true">⚠</span> {dobErr}
    </p>
  )}
</div>
```

### Client — Plain HTML

```html
<label for="dob">Date of birth <span aria-hidden="true">*</span></label>
<input
  type="date"
  id="dob"
  name="dob"
  autocomplete="bday"
  required
  aria-required="true"
  min="1905-01-01"
  max="2012-12-31"
  style="font-size:16px;min-height:44px;width:100%">
<p id="dob-help">You must be at least 13 years old.</p>
<p id="dob-error" role="alert" class="error" hidden></p>
<!-- Compute min/max dynamically on page load if precision matters -->
```

### Fallback: 3-select pattern (Day / Month / Year)
Some teams prefer 3 selects (older-audience products, accessibility regulators
that flag native pickers, browsers without good `type=date` support).

```tsx
<fieldset>
  <legend>Date of birth <span aria-hidden="true">*</span></legend>
  <div className="flex gap-2">
    <input  type="number" inputMode="numeric" autoComplete="bday-day"   min={1} max={31} aria-label="Day"   className="w-20 min-h-[44px]" />
    <select autoComplete="bday-month" aria-label="Month" className="min-h-[44px]"> {/* 12 options */} </select>
    <input  type="number" inputMode="numeric" autoComplete="bday-year"  min={1905} max={new Date().getFullYear()} aria-label="Year" className="w-28 min-h-[44px]" />
  </div>
</fieldset>
```

Combine into `YYYY-MM-DD` before submit and run the same validation.

### Server

```ts
const dobRaw = sanitizeInput(body.dob, 10);
const m = /^(\d{4})-(\d{2})-(\d{2})$/.exec(dobRaw);
if (!m) {
  return NextResponse.json({ error: "Date of birth is required (YYYY-MM-DD)." }, { status: 400 });
}
const [, ys, ms, ds] = m;
const y = Number(ys), mo = Number(ms), d = Number(ds);
const dob = new Date(Date.UTC(y, mo - 1, d));
const now = new Date();

// Reject malformed (e.g. 2024-02-31 round-trips to a different date)
if (dob.getUTCFullYear() !== y || dob.getUTCMonth() !== mo - 1 || dob.getUTCDate() !== d || dob > now) {
  return NextResponse.json({ error: "Invalid date of birth." }, { status: 400 });
}

let age = now.getUTCFullYear() - y;
if (now.getUTCMonth() < mo - 1 || (now.getUTCMonth() === mo - 1 && now.getUTCDate() < d)) age--;

const MIN_AGE = 13; // adjust per Phase 1 Q12
if (age < MIN_AGE || age > 120) {
  return NextResponse.json({ error: `You must be at least ${MIN_AGE} years old.` }, { status: 400 });
}
// Store dobRaw (YYYY-MM-DD); compute age at query time, never cache it.
```

**Why not `new Date(userString)`:** ambiguous formats (`02/03/2024` is March 2
in EU and Feb 3 in US), implicit local-timezone parsing, and silent NaN on
invalid input. Always parse explicitly as UTC.

**Why year/month/day age comparison (not ms-divide):** dividing by
`365.25 * 24 * 3600 * 1000` is off by a day every leap year — a 17-year-old
born on Feb 29 can become "18" a day early or late.

### DOB error copy

| Situation | Copy |
|-----------|------|
| Empty + required | "Date of birth is required." |
| Fails format / non-existent date (e.g. 2024-02-31) | "Enter a real date in the past." |
| Future date | "Date of birth can't be in the future." |
| Under min age | "You must be at least 13 years old." (substitute) |
| > 120 years | "Please enter a valid date of birth." |

---

## 5. Cross-field rules

| Situation | Rule |
|-----------|------|
| Form has both `phone` and `email`, neither marked required individually | Require **at least one** — server-side check after sanitize. Surface as a single error attached to the email field. |
| `preferred_contact = "phone"` (radio/pill) | Make `phone` required dynamically; clear when switched back to email |
| `preferred_contact = "email"` | Make `email` required (already the default) |
| Multiple URL fields (e.g. `linkedin` + `portfolio`) | Validate each independently with its own allow-list; no "at least one" coupling unless the user explicitly asks |
| DOB + age-gated content | Block submission and show the gate copy *before* the API call; do not send the submission, do not store the date |

---

## Summary: per-field minimum bar

| Field | `type` | `inputmode` | `autocomplete` | Server check |
|-------|--------|-------------|----------------|--------------|
| Phone | `tel`  | `tel`       | `tel`          | `parsePhoneNumberFromString().isValid()` → store `.number` (E.164) |
| Email | `email`| `email`     | `email`        | Existing `EMAIL_RE` + optional disposable list |
| URL   | `url`  | `url`       | `url`          | `new URL()` + protocol allow-list (`http`, `https` only) + optional hostname allow-list |
| DOB   | `date` | —           | `bday`         | UTC `Date.UTC` parse + year/month/day age check + `[minAge, 120]` range |

If a generated form misses any cell in this table, it fails the Phase A
audit at P1 (accessibility / correctness).
