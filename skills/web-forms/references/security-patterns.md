# Security Patterns

Copy-paste-ready code for production-safe form handling.
Read this file in **Phase 4** (form component security layer) and **Phase 5** (API route security layer).

Every generated form includes all **Default** tier items automatically.
**reCAPTCHA v3** is included when the user answers Yes to Phase 1 Q9, or for any public-facing form where Q9 was not explicitly answered No.

---

## Utilities (add to `utils/` in the project)

### `escapeHtml` — XSS prevention

Wrap every `${value}` that lands inside an HTML string in an email body.

```ts
// utils/escapeHtml.ts
export function escapeHtml(str: unknown): string {
  return String(str ?? "")
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#x27;");
}
```

Usage: `escapeHtml(name)`, `escapeHtml(v)`, etc. — applied to every user-supplied value in email HTML.

---

### `sanitizeInput` — CRLF strip + length cap

Strips carriage-return / line-feed characters (prevents email header injection) and enforces a max length.

```ts
// utils/sanitizeInput.ts
export function sanitizeInput(value: unknown, maxLen = 1000): string {
  return String(value ?? "")
    .replace(/[\r\n]/g, " ")   // CRLF strip — prevents header injection
    .trim()
    .slice(0, maxLen);
}
```

**Recommended max lengths:**

| Field | `maxLen` |
|-------|----------|
| `name` | 100 |
| `email` | 254 (RFC 5321) |
| `subject` / single-line | 200 |
| `message` / `textarea` | 5000 |
| `company`, `role`, `title` | 150 |

---

## API Route Security — Standard Validation Block

Replace the minimal `if (!name || !email)` check in every route with this full block.
Add utility imports at the top of the route file:

```ts
import { escapeHtml }     from "@/utils/escapeHtml";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";
```

### Body size guard (place before `req.json()`)

```ts
const raw = await req.text();
if (raw.length > 50_000) {
  return NextResponse.json({ error: "Request too large." }, { status: 413 });
}
const body = JSON.parse(raw);
```

For `multipart/form-data` routes (file upload forms), the existing file size guard covers this path — skip the body-size guard and keep `req.formData()`.

### Full validation + bot-guard block

```ts
// 1. Sanitize inputs
const name    = sanitizeInput(body.name,    100);
const email   = sanitizeInput(body.email,   254);
const message = sanitizeInput(body.message, 5000);
// sanitize additional preset fields similarly, e.g.:
// const company = sanitizeInput(body.company, 150);

// 2. Required field check
if (!name || !email) {
  return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
}

// 3. Email format validation
const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
if (!EMAIL_RE.test(email)) {
  return NextResponse.json({ error: "Invalid email address." }, { status: 400 });
}

// 4. Bot / abuse guards (all silent-200 — never tip off bots)
const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim()
         ?? req.headers.get("x-real-ip")
         ?? "unknown";

if (!checkRateLimit(ip)) {
  return NextResponse.json({ error: "Too many requests. Please wait a moment." }, { status: 429 });
}
if (body.website) {
  return NextResponse.json({ success: true }); // honeypot triggered
}
if (!body._t || (Date.now() - Number(body._t)) < 3_000) {
  return NextResponse.json({ success: true }); // time guard — submitted too fast
}
if (isDuplicate(email)) {
  return NextResponse.json({ success: true }); // duplicate submission
}
if (isSpam(message)) {
  return NextResponse.json({ success: true }); // spam keyword detected
}

// reCAPTCHA v3 — uncomment when Q9 = Yes:
// const captchaOk = await verifyRecaptcha(body.recaptchaToken ?? "");
// if (!captchaOk) return NextResponse.json({ success: true });
```

---

## CORS Origin Check (optional but recommended)

Add at the very top of the POST handler, before the body read:

```ts
const origin  = req.headers.get("origin") ?? "";
const ALLOWED = (process.env.ALLOWED_ORIGINS ?? "").split(",").map(s => s.trim()).filter(Boolean);
if (ALLOWED.length > 0 && !ALLOWED.includes(origin)) {
  return NextResponse.json({ error: "Forbidden." }, { status: 403 });
}
```

Env var: `ALLOWED_ORIGINS="https://yourdomain.com"`. If unset, the check is skipped (dev-friendly).

---

## Rate Limiter

### Default — In-memory Map (single Next.js instance)

Works for dev, single-server production, and single-region Vercel deployments.
Resets on cold start — acceptable for most contact form traffic.

```ts
// utils/rateLimiter.ts
const ipMap     = new Map<string, { count: number; reset: number }>();
const RATE_LIMIT = 5;
const WINDOW_MS  = 60_000; // 60 seconds

export function checkRateLimit(ip: string): boolean {
  const now   = Date.now();
  const entry = ipMap.get(ip);
  if (!entry || now > entry.reset) {
    ipMap.set(ip, { count: 1, reset: now + WINDOW_MS });
    return true; // allowed
  }
  if (entry.count >= RATE_LIMIT) return false; // blocked
  entry.count++;
  return true;
}

// Periodic cleanup to prevent memory leak
setInterval(() => {
  const now = Date.now();
  for (const [ip, entry] of ipMap) {
    if (now > entry.reset) ipMap.delete(ip);
  }
}, WINDOW_MS * 2);
```

### Production upgrade — Upstash Redis (multi-instance / serverless)

Use this when deploying to Vercel functions, multiple server instances, or Edge runtime.

```bash
npm install @upstash/ratelimit @upstash/redis
```

```ts
// utils/rateLimiter.upstash.ts
import { Ratelimit } from "@upstash/ratelimit";
import { Redis }     from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis:     Redis.fromEnv(),
  limiter:   Ratelimit.slidingWindow(5, "60 s"),
  analytics: true,
});

export async function checkRateLimitUpstash(ip: string): Promise<boolean> {
  const { success } = await ratelimit.limit(ip);
  return success;
}
```

Env vars:
```
UPSTASH_REDIS_REST_URL=https://...
UPSTASH_REDIS_REST_TOKEN=...
```

---

## Honeypot — Client HTML + Server Check

A hidden field that real users never see or interact with. Bots auto-fill it.
Server silently returns `200 { success: true }` — never a `400` that reveals the guard.

### React / TSX version

```tsx
{/* Honeypot — add just before the submit button */}
<div
  aria-hidden="true"
  style={{ position: "absolute", left: "-9999px", height: 0, overflow: "hidden" }}
>
  <label htmlFor="website">Leave this blank</label>
  <input
    type="text"
    id="website"
    name="website"
    tabIndex={-1}
    autoComplete="off"
    value={form.website ?? ""}
    onChange={e => setForm(prev => ({ ...prev, website: e.target.value }))}
  />
</div>
```

Add `website: ""` to the `FormData` type and `INITIAL` state object.

### Plain HTML version

```html
<!-- Honeypot — add just before the submit button -->
<div aria-hidden="true" style="position:absolute;left:-9999px;height:0;overflow:hidden">
  <label for="website">Leave this blank</label>
  <input type="text" id="website" name="website" tabindex="-1" autocomplete="off" value="">
</div>
```

### Server check

```ts
if (body.website) {
  return NextResponse.json({ success: true }); // silent pass
}
```

---

## Submission Time Guard

Real users take at least 3–10 seconds to fill a form. Bots submit in milliseconds.

### React / TSX — record load time in state

```tsx
const [formLoadTime] = useState(() => Date.now());

// Include in the fetch body:
body: JSON.stringify({ ...form, _t: formLoadTime }),
```

### Plain HTML — hidden field populated on load

```html
<input type="hidden" id="_t" name="_t">
<script>document.getElementById('_t').value = Date.now();</script>
```

### Server check (silent pass — never expose the threshold)

```ts
const MIN_FILL_MS = 3_000;
if (!body._t || (Date.now() - Number(body._t)) < MIN_FILL_MS) {
  return NextResponse.json({ success: true });
}
```

---

## Duplicate Submission Detection

Blocks the same email address from submitting again within a 5-minute window.
Pairs with rate limiting: rate limiter blocks by IP, duplicate guard blocks by email.

```ts
// utils/duplicateGuard.ts
const seen  = new Map<string, number>(); // email → expiry timestamp
const TTL_MS = 5 * 60_000;              // 5 minutes

export function isDuplicate(email: string): boolean {
  const now = Date.now();
  const exp = seen.get(email);
  if (exp && now < exp) return true; // duplicate
  seen.set(email, now + TTL_MS);
  return false;
}

setInterval(() => {
  const now = Date.now();
  for (const [k, exp] of seen) if (now > exp) seen.delete(k);
}, TTL_MS);
```

Usage: `if (isDuplicate(email)) return NextResponse.json({ success: true });`

---

## Spam Keyword Filter

Configurable server-side keyword list checked against the message field.
Keep the list short and conservative — false positives lose real leads.

```ts
// utils/spamFilter.ts
const SPAM_KEYWORDS = [
  "casino",
  "viagra",
  "crypto investment",
  "binary options",
  "click here to claim",
  "you have won",
  "wire transfer",
  // extend as needed
];

export function isSpam(text: string): boolean {
  const lower = text.toLowerCase();
  return SPAM_KEYWORDS.some(kw => lower.includes(kw));
}
```

Usage: `if (isSpam(message)) return NextResponse.json({ success: true });`

---

## reCAPTCHA v3 — Invisible CAPTCHA (no checkbox, no friction)

Include when Q9 = Yes (or for any high-traffic public form).
Google scores the user silently: score ≥ 0.5 = likely human, < 0.5 = likely bot.

```bash
npm install react-google-recaptcha-v3
```

**Env vars:**
```
NEXT_PUBLIC_RECAPTCHA_SITE_KEY=6Le...     # public — safe to expose
RECAPTCHA_SECRET_KEY=6Le...               # server-only — never expose
```

### Client — wrap the app

In `layout.tsx` (App Router) or `_app.tsx` (Pages Router):

```tsx
import { GoogleReCaptchaProvider } from "react-google-recaptcha-v3";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <GoogleReCaptchaProvider reCaptchaKey={process.env.NEXT_PUBLIC_RECAPTCHA_SITE_KEY!}>
          {children}
        </GoogleReCaptchaProvider>
      </body>
    </html>
  );
}
```

### Client — execute on form submit

```tsx
import { useGoogleReCaptcha } from "react-google-recaptcha-v3";

const { executeRecaptcha } = useGoogleReCaptcha();

const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  if (!executeRecaptcha) return;

  const token = await executeRecaptcha("contact_form");

  const res = await fetch("/api/contact", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ ...form, recaptchaToken: token, _t: formLoadTime }),
  });
  // ...
};
```

### Server — verify token (add after bot guards, before email sends)

```ts
async function verifyRecaptcha(token: string): Promise<boolean> {
  const res = await fetch("https://www.google.com/recaptcha/api/siteverify", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: `secret=${process.env.RECAPTCHA_SECRET_KEY}&response=${token}`,
  });
  const data = await res.json();
  return data.success === true && data.score >= 0.5;
}

// In POST handler, after duplicate/spam guards:
const captchaOk = await verifyRecaptcha(body.recaptchaToken ?? "");
if (!captchaOk) {
  return NextResponse.json({ success: true }); // silent pass
}
```

### reCAPTCHA setup checklist
1. Go to [Google reCAPTCHA Admin](https://www.google.com/recaptcha/admin/create)
2. Choose **reCAPTCHA v3**
3. Add your domain(s)
4. Copy **Site Key** → `NEXT_PUBLIC_RECAPTCHA_SITE_KEY`
5. Copy **Secret Key** → `RECAPTCHA_SECRET_KEY` (server only — never prefix with `NEXT_PUBLIC_`)

---

## Security Checklist Block (output in Phase 6)

Always append this to the post-generation checklist:

```
🔒 Security defaults included:
   ✅ escapeHtml — XSS prevention in email HTML
   ✅ sanitizeInput — CRLF strip + length caps
   ✅ Email regex validation + body size guard (50 KB)
   ✅ Honeypot field
   ✅ Submission time guard (3s minimum)
   ✅ In-memory rate limit (5 req / 60s / IP)
   ✅ Duplicate email guard (5-min window)
   ✅ Spam keyword filter
   [ ] reCAPTCHA v3 — set NEXT_PUBLIC_RECAPTCHA_SITE_KEY + RECAPTCHA_SECRET_KEY
       npm install react-google-recaptcha-v3
   [ ] Upgrade rate limiter to Upstash for multi-instance / serverless deployments
       npm install @upstash/ratelimit @upstash/redis
       set UPSTASH_REDIS_REST_URL + UPSTASH_REDIS_REST_TOKEN
```

And add these to the `.env.local` block:
```
ALLOWED_ORIGINS="https://yourdomain.com"
# NEXT_PUBLIC_RECAPTCHA_SITE_KEY="6Le..."   ← reCAPTCHA v3 (Q9 = Yes)
# RECAPTCHA_SECRET_KEY="6Le..."             ← reCAPTCHA v3 (Q9 = Yes)
```
