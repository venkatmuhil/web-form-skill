# Email Services Integration Guide

Integration code for all supported email services. Each section covers:
- Install command
- Environment variables
- API route (Next.js App Router) — adapt for other frameworks as needed
- Brevo also covers CRM contact sync

---

## Brevo (recommended default)

**Best for:** CRM + email in one; contact list sync; EU data residency; generous free tier
**Free tier:** 300 emails/day, unlimited contacts
**Docs:** https://developers.brevo.com

### Install
```bash
npm install @getbrevo/brevo
```

### Env vars
```bash
BREVO_API_KEY=your_key_here
SENDER_NAME="Your Company"
SENDER_EMAIL="hello@yourdomain.com"   # must be verified in Brevo
NOTIFY_EMAIL="team@yourdomain.com"
BREVO_LIST_ID=3                        # optional — for contact sync
```

### API Route (Next.js App Router)
```ts
// src/app/api/contact/route.ts
import { NextRequest, NextResponse } from "next/server";
import * as Brevo from "@getbrevo/brevo";
import { escapeHtml }     from "@/utils/escapeHtml";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";

const apiInstance = new Brevo.TransactionalEmailsApi();
apiInstance.setApiKey(Brevo.TransactionalEmailsApiApiKeys.apiKey, process.env.BREVO_API_KEY!);

const contactsApi = new Brevo.ContactsApi();
contactsApi.setApiKey(Brevo.ContactsApiApiKeys.apiKey, process.env.BREVO_API_KEY!);

export async function POST(req: NextRequest) {
  // Body size guard — reject before JSON.parse
  const raw = await req.text();
  if (raw.length > 50_000) {
    return NextResponse.json({ error: "Request too large." }, { status: 413 });
  }
  const body = JSON.parse(raw);

  // Sanitize inputs — CRLF strip + length caps
  const name    = sanitizeInput(body.name,    100);
  const email   = sanitizeInput(body.email,   254);
  const message = sanitizeInput(body.message, 5000);
  // sanitize additional preset fields similarly, e.g.:
  // const company = sanitizeInput(body.company, 150);
  const rest = Object.fromEntries(
    Object.entries(body)
      .filter(([k]) => !["name","email","message","website","_t","recaptchaToken"].includes(k))
      .map(([k, v]) => [k, Array.isArray(v) ? v.map(s => sanitizeInput(s, 200)) : sanitizeInput(v, 200)])
  );

  // Required field + email format validation
  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }
  const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
  if (!EMAIL_RE.test(email)) {
    return NextResponse.json({ error: "Invalid email address." }, { status: 400 });
  }

  // Bot / abuse guards (all silent-200 — never tip off bots)
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim()
           ?? req.headers.get("x-real-ip")
           ?? "unknown";
  if (!checkRateLimit(ip)) {
    return NextResponse.json({ error: "Too many requests. Please wait a moment." }, { status: 429 });
  }
  if (body.website)                                              return NextResponse.json({ success: true });
  if (!body._t || (Date.now() - Number(body._t)) < 3_000)       return NextResponse.json({ success: true });
  if (isDuplicate(email))                                        return NextResponse.json({ success: true });
  if (isSpam(message))                                           return NextResponse.json({ success: true });

  // reCAPTCHA v3 — uncomment when Q9 = Yes:
  // const captchaOk = await verifyRecaptcha(body.recaptchaToken ?? "");
  // if (!captchaOk) return NextResponse.json({ success: true });
  // (see references/security-patterns.md → reCAPTCHA v3 for verifyRecaptcha implementation)

  // Build submission table — escapeHtml on all user values
  const rows = [
    ["Name", name], ["Email", email], ["Message", message],
    ...Object.entries(rest).map(([k, v]) => [
      k.replace(/_/g, " ").replace(/\b\w/g, c => c.toUpperCase()),
      Array.isArray(v) ? v.join(", ") : String(v),
    ]),
  ];
  const tableRows = rows
    .map(([k, v]) => `<tr><td style="padding:6px 12px;font-weight:600">${escapeHtml(k)}</td><td style="padding:6px 12px">${escapeHtml(v)}</td></tr>`)
    .join("");

  // Notify team
  await apiInstance.sendTransacEmail({
    sender: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    to: [{ email: process.env.NOTIFY_EMAIL! }],
    replyTo: { email, name },
    subject: `New enquiry from ${sanitizeInput(name, 100)}`,
    htmlContent: `
      <h2 style="font-family:sans-serif">New Contact Form Submission</h2>
      <table style="font-family:sans-serif;border-collapse:collapse">${tableRows}</table>
    `,
  });

  // Confirm submitter
  await apiInstance.sendTransacEmail({
    sender: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    to: [{ email, name }],
    subject: "We received your message!",
    htmlContent: `
      <p style="font-family:sans-serif">Hi ${escapeHtml(name)},</p>
      <p style="font-family:sans-serif">Thanks for reaching out — we'll be in touch shortly.</p>
      <p style="font-family:sans-serif">— The Team</p>
    `,
  });

  // Sync contact to Brevo CRM list (optional)
  if (process.env.BREVO_LIST_ID) {
    await contactsApi.createContact({
      email,
      attributes: { FIRSTNAME: name.split(" ")[0], LASTNAME: name.split(" ").slice(1).join(" ") },
      listIds: [Number(process.env.BREVO_LIST_ID)],
      updateEnabled: true,
    });
  }

  return NextResponse.json({ success: true });
}
```

### Brevo setup checklist
1. Sign up at [brevo.com](https://www.brevo.com)
2. **Settings → SMTP & API → API Keys** → create key
3. **Settings → Senders & IP → Senders** → add + verify sending domain
4. *(Optional)* **Contacts → Lists** → create list → note the ID for `BREVO_LIST_ID`

---

## Resend

**Best for:** Developer-friendly API; React Email template support; clean DX
**Free tier:** 100 emails/day, 3,000/month
**Docs:** https://resend.com/docs

### Install
```bash
npm install resend
```

### Env vars
```bash
RESEND_API_KEY=re_xxxxxxxxxxxx
SENDER_NAME="Your Company"
SENDER_EMAIL="hello@yourdomain.com"   # must be from a verified domain
NOTIFY_EMAIL="team@yourdomain.com"
```

### API Route (Next.js App Router)
```ts
// src/app/api/contact/route.ts
import { NextRequest, NextResponse } from "next/server";
import { Resend } from "resend";
import { escapeHtml }     from "@/utils/escapeHtml";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";

const resend = new Resend(process.env.RESEND_API_KEY);

export async function POST(req: NextRequest) {
  // Body size guard
  const raw = await req.text();
  if (raw.length > 50_000) {
    return NextResponse.json({ error: "Request too large." }, { status: 413 });
  }
  const body = JSON.parse(raw);

  // Sanitize inputs
  const name    = sanitizeInput(body.name,    100);
  const email   = sanitizeInput(body.email,   254);
  const message = sanitizeInput(body.message, 5000);
  const rest = Object.fromEntries(
    Object.entries(body)
      .filter(([k]) => !["name","email","message","website","_t","recaptchaToken"].includes(k))
      .map(([k, v]) => [k, Array.isArray(v) ? v.map(s => sanitizeInput(s, 200)) : sanitizeInput(v, 200)])
  );

  // Required field + email format validation
  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }
  const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
  if (!EMAIL_RE.test(email)) {
    return NextResponse.json({ error: "Invalid email address." }, { status: 400 });
  }

  // Bot / abuse guards
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim()
           ?? req.headers.get("x-real-ip")
           ?? "unknown";
  if (!checkRateLimit(ip)) {
    return NextResponse.json({ error: "Too many requests. Please wait a moment." }, { status: 429 });
  }
  if (body.website)                                              return NextResponse.json({ success: true });
  if (!body._t || (Date.now() - Number(body._t)) < 3_000)       return NextResponse.json({ success: true });
  if (isDuplicate(email))                                        return NextResponse.json({ success: true });
  if (isSpam(message))                                           return NextResponse.json({ success: true });

  // reCAPTCHA v3 — uncomment when Q9 = Yes:
  // const captchaOk = await verifyRecaptcha(body.recaptchaToken ?? "");
  // if (!captchaOk) return NextResponse.json({ success: true });

  // Build submission table — escapeHtml on all user values
  const rows = [
    ["Name", name], ["Email", email], ["Message", message],
    ...Object.entries(rest).map(([k, v]) => [
      k.replace(/_/g, " ").replace(/\b\w/g, c => c.toUpperCase()),
      Array.isArray(v) ? v.join(", ") : String(v),
    ]),
  ];
  const tableHtml = rows
    .map(([k, v]) => `<tr><td style="padding:6px 12px;font-weight:600">${escapeHtml(k)}</td><td style="padding:6px 12px">${escapeHtml(v)}</td></tr>`)
    .join("");

  // Notify team
  await resend.emails.send({
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [process.env.NOTIFY_EMAIL!],
    reply_to: email,
    subject: `New enquiry from ${sanitizeInput(name, 100)}`,
    html: `<h2>New Form Submission</h2><table style="border-collapse:collapse">${tableHtml}</table>`,
  });

  // Confirm submitter
  await resend.emails.send({
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [email],
    subject: "We received your message!",
    html: `<p>Hi ${escapeHtml(name)},</p><p>Thanks for reaching out — we'll be in touch shortly.</p><p>— The Team</p>`,
  });

  return NextResponse.json({ success: true });
}
```

### Resend setup checklist
1. Sign up at [resend.com](https://resend.com)
2. **API Keys** → create key
3. **Domains** → add your domain → add DNS records (SPF, DKIM)
4. Once verified, use `you@yourdomain.com` as sender

---

## SendGrid (Twilio)

**Best for:** High-volume sending; enterprise; strong deliverability analytics
**Free tier:** 100 emails/day
**Docs:** https://docs.sendgrid.com

### Install
```bash
npm install @sendgrid/mail
```

### Env vars
```bash
SENDGRID_API_KEY=SG.xxxxxxxxxxxx
SENDER_NAME="Your Company"
SENDER_EMAIL="hello@yourdomain.com"
NOTIFY_EMAIL="team@yourdomain.com"
```

### API Route (Next.js App Router)
```ts
// src/app/api/contact/route.ts
import { NextRequest, NextResponse } from "next/server";
import sgMail from "@sendgrid/mail";
import { escapeHtml }     from "@/utils/escapeHtml";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";

sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

export async function POST(req: NextRequest) {
  // Body size guard
  const raw = await req.text();
  if (raw.length > 50_000) {
    return NextResponse.json({ error: "Request too large." }, { status: 413 });
  }
  const body = JSON.parse(raw);

  // Sanitize inputs
  const name    = sanitizeInput(body.name,    100);
  const email   = sanitizeInput(body.email,   254);
  const message = sanitizeInput(body.message, 5000);
  const rest = Object.fromEntries(
    Object.entries(body)
      .filter(([k]) => !["name","email","message","website","_t","recaptchaToken"].includes(k))
      .map(([k, v]) => [k, Array.isArray(v) ? v.map(s => sanitizeInput(s, 200)) : sanitizeInput(v, 200)])
  );

  // Required field + email format validation
  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }
  const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
  if (!EMAIL_RE.test(email)) {
    return NextResponse.json({ error: "Invalid email address." }, { status: 400 });
  }

  // Bot / abuse guards
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim()
           ?? req.headers.get("x-real-ip")
           ?? "unknown";
  if (!checkRateLimit(ip)) {
    return NextResponse.json({ error: "Too many requests. Please wait a moment." }, { status: 429 });
  }
  if (body.website)                                              return NextResponse.json({ success: true });
  if (!body._t || (Date.now() - Number(body._t)) < 3_000)       return NextResponse.json({ success: true });
  if (isDuplicate(email))                                        return NextResponse.json({ success: true });
  if (isSpam(message))                                           return NextResponse.json({ success: true });

  // reCAPTCHA v3 — uncomment when Q9 = Yes:
  // const captchaOk = await verifyRecaptcha(body.recaptchaToken ?? "");
  // if (!captchaOk) return NextResponse.json({ success: true });

  // Build submission table — escapeHtml on all user values
  const rows = Object.entries({ name, email, message, ...rest })
    .map(([k, v]) => `<tr><td><strong>${escapeHtml(k)}</strong></td><td>${escapeHtml(Array.isArray(v) ? v.join(", ") : String(v))}</td></tr>`)
    .join("");

  // Notify team
  await sgMail.send({
    to: process.env.NOTIFY_EMAIL!,
    from: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    replyTo: email,
    subject: `New enquiry from ${sanitizeInput(name, 100)}`,
    html: `<table>${rows}</table>`,
  });

  // Confirm submitter
  await sgMail.send({
    to: email,
    from: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    subject: "We received your message!",
    html: `<p>Hi ${escapeHtml(name)}, thanks for reaching out — we'll be in touch shortly.</p>`,
  });

  return NextResponse.json({ success: true });
}
```

### SendGrid setup checklist
1. Sign up at [sendgrid.com](https://sendgrid.com)
2. **Settings → API Keys** → create key with "Mail Send" permission
3. **Settings → Sender Authentication** → verify domain (recommended) or single sender

---

## Mailgun

**Best for:** Pay-as-you-go; good EU data residency option (eu.mailgun.com region)
**Free tier:** Trial only (limited sends); then pay-as-you-go
**Docs:** https://documentation.mailgun.com

### Install
```bash
npm install mailgun.js form-data
```

### Env vars
```bash
MAILGUN_API_KEY=key-xxxxxxxxxxxx
MAILGUN_DOMAIN=mg.yourdomain.com    # your Mailgun sending domain
SENDER_NAME="Your Company"
SENDER_EMAIL="hello@mg.yourdomain.com"
NOTIFY_EMAIL="team@yourdomain.com"
# For EU: MAILGUN_REGION=eu
```

### API Route (Next.js App Router)
```ts
import { NextRequest, NextResponse } from "next/server";
import Mailgun from "mailgun.js";
import FormData from "form-data";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";

const mg = new Mailgun(FormData).client({
  username: "api",
  key: process.env.MAILGUN_API_KEY!,
  url: process.env.MAILGUN_REGION === "eu"
    ? "https://api.eu.mailgun.net"
    : "https://api.mailgun.net",
});

export async function POST(req: NextRequest) {
  // Body size guard
  const raw = await req.text();
  if (raw.length > 50_000) {
    return NextResponse.json({ error: "Request too large." }, { status: 413 });
  }
  const body = JSON.parse(raw);

  // Sanitize inputs (CRLF strip is critical for Mailgun — h:Reply-To header injection risk)
  const name    = sanitizeInput(body.name,    100);
  const email   = sanitizeInput(body.email,   254);
  const message = sanitizeInput(body.message, 5000);
  const rest = Object.fromEntries(
    Object.entries(body)
      .filter(([k]) => !["name","email","message","website","_t","recaptchaToken"].includes(k))
      .map(([k, v]) => [k, Array.isArray(v) ? v.map(s => sanitizeInput(s, 200)) : sanitizeInput(v, 200)])
  );

  // Required field + email format validation
  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }
  const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
  if (!EMAIL_RE.test(email)) {
    return NextResponse.json({ error: "Invalid email address." }, { status: 400 });
  }

  // Bot / abuse guards
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim()
           ?? req.headers.get("x-real-ip")
           ?? "unknown";
  if (!checkRateLimit(ip)) {
    return NextResponse.json({ error: "Too many requests. Please wait a moment." }, { status: 429 });
  }
  if (body.website)                                              return NextResponse.json({ success: true });
  if (!body._t || (Date.now() - Number(body._t)) < 3_000)       return NextResponse.json({ success: true });
  if (isDuplicate(email))                                        return NextResponse.json({ success: true });
  if (isSpam(message))                                           return NextResponse.json({ success: true });

  // reCAPTCHA v3 — uncomment when Q9 = Yes:
  // const captchaOk = await verifyRecaptcha(body.recaptchaToken ?? "");
  // if (!captchaOk) return NextResponse.json({ success: true });

  // Build plain-text notification (Mailgun default — safe from HTML injection)
  const text = Object.entries({ name, email, message, ...rest })
    .map(([k, v]) => `${k}: ${Array.isArray(v) ? v.join(", ") : v}`)
    .join("\n");

  // Notify team
  await mg.messages.create(process.env.MAILGUN_DOMAIN!, {
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [process.env.NOTIFY_EMAIL!],
    "h:Reply-To": sanitizeInput(email, 254), // belt-and-suspenders: already sanitized above
    subject: `New enquiry from ${sanitizeInput(name, 100)}`,
    text,
  });

  // Confirm submitter
  await mg.messages.create(process.env.MAILGUN_DOMAIN!, {
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [email],
    subject: "We received your message!",
    text: `Hi ${sanitizeInput(name, 100)},\n\nThanks for reaching out — we'll be in touch shortly.\n\n— The Team`,
  });

  return NextResponse.json({ success: true });
}
```

### Mailgun setup checklist
1. Sign up at [mailgun.com](https://www.mailgun.com)
2. **Sending → Domains** → add domain → add DNS records
3. **API Security** → get API key
4. Choose region: US (`api.mailgun.net`) or EU (`api.eu.mailgun.net`)

---

## Postmark

**Best for:** Best-in-class deliverability for transactional email; strict no-bulk policy
**Free tier:** Trial (100 test emails); then paid
**Docs:** https://postmarkapp.com/developer

### Install
```bash
npm install postmark
```

### Env vars
```bash
POSTMARK_SERVER_TOKEN=xxxxxxxxxxxx
SENDER_NAME="Your Company"
SENDER_EMAIL="hello@yourdomain.com"   # must match verified Sender Signature
NOTIFY_EMAIL="team@yourdomain.com"
```

### API Route (Next.js App Router)
```ts
import { NextRequest, NextResponse } from "next/server";
import * as postmark from "postmark";
import { escapeHtml }     from "@/utils/escapeHtml";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";

const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN!);

export async function POST(req: NextRequest) {
  // Body size guard
  const raw = await req.text();
  if (raw.length > 50_000) {
    return NextResponse.json({ error: "Request too large." }, { status: 413 });
  }
  const body = JSON.parse(raw);

  // Sanitize inputs
  const name    = sanitizeInput(body.name,    100);
  const email   = sanitizeInput(body.email,   254);
  const message = sanitizeInput(body.message, 5000);
  const rest = Object.fromEntries(
    Object.entries(body)
      .filter(([k]) => !["name","email","message","website","_t","recaptchaToken"].includes(k))
      .map(([k, v]) => [k, Array.isArray(v) ? v.map(s => sanitizeInput(s, 200)) : sanitizeInput(v, 200)])
  );

  // Required field + email format validation
  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }
  const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
  if (!EMAIL_RE.test(email)) {
    return NextResponse.json({ error: "Invalid email address." }, { status: 400 });
  }

  // Bot / abuse guards
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim()
           ?? req.headers.get("x-real-ip")
           ?? "unknown";
  if (!checkRateLimit(ip)) {
    return NextResponse.json({ error: "Too many requests. Please wait a moment." }, { status: 429 });
  }
  if (body.website)                                              return NextResponse.json({ success: true });
  if (!body._t || (Date.now() - Number(body._t)) < 3_000)       return NextResponse.json({ success: true });
  if (isDuplicate(email))                                        return NextResponse.json({ success: true });
  if (isSpam(message))                                           return NextResponse.json({ success: true });

  // reCAPTCHA v3 — uncomment when Q9 = Yes:
  // const captchaOk = await verifyRecaptcha(body.recaptchaToken ?? "");
  // if (!captchaOk) return NextResponse.json({ success: true });

  // Build submission table — escapeHtml on all user values
  const rows = Object.entries({ name, email, message, ...rest })
    .map(([k, v]) => `<tr><td><strong>${escapeHtml(k)}</strong></td><td>${escapeHtml(Array.isArray(v) ? v.join(", ") : String(v))}</td></tr>`)
    .join("");

  // Notify team
  await client.sendEmail({
    From: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    To: process.env.NOTIFY_EMAIL!,
    ReplyTo: email,
    Subject: `New enquiry from ${sanitizeInput(name, 100)}`,
    HtmlBody: `<table>${rows}</table>`,
    MessageStream: "outbound",
  });

  // Confirm submitter
  await client.sendEmail({
    From: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    To: email,
    Subject: "We received your message!",
    HtmlBody: `<p>Hi ${escapeHtml(name)},</p><p>Thanks for reaching out — we'll be in touch shortly.</p>`,
    MessageStream: "outbound",
  });

  return NextResponse.json({ success: true });
}
```

### Postmark setup checklist
1. Sign up at [postmarkapp.com](https://postmarkapp.com)
2. Create a **Server** → get Server API Token
3. **Sender Signatures** → add + verify sender email/domain
4. Note: Postmark is transactional-only — don't use it for bulk marketing email

---

## No 3rd-Party Email Service

Three production-ready handlers when the user does not want to depend on Brevo / Resend / SendGrid / Mailgun / Postmark. Each one slots into the same Phase 5 API-route security contract — only the **delivery step** changes.

Pick one (or combine — DB + Slack is a common pairing for "no email at all"):

| Option | When it fits | Runs on serverless? |
|--------|--------------|---------------------|
| SMTP / Nodemailer | Has own mail server, Google Workspace, Microsoft 365 SMTP, or any SMTP relay | Yes (use a relay, not local sendmail) |
| Database persistence | Wants every submission stored, queried, exported | Yes (with a hosted DB) |
| Slack / Discord / Teams webhook | Team lives in chat; no inbox needed | Yes |
| Generic outbound webhook | Forwarding to Zapier / Make / n8n / own backend | Yes |

The original generic-webhook stub is at the bottom of this section.

---

### Option 1 — SMTP via Nodemailer

For users with their own mail server, Google Workspace, Microsoft 365, Amazon SES SMTP, or any SMTP relay. No 3rd-party SDK required.

#### Install
```bash
npm install nodemailer
npm install --save-dev @types/nodemailer
```

#### Transport — `utils/mailer.ts` (singleton, reused across requests)
```ts
import nodemailer from "nodemailer";

declare global { var __mailer: nodemailer.Transporter | undefined; }

export const mailer =
  global.__mailer ??
  nodemailer.createTransport({
    host:   process.env.SMTP_HOST,
    port:   Number(process.env.SMTP_PORT ?? 587),
    secure: process.env.SMTP_SECURE === "true",   // true for 465, false for 587/STARTTLS
    auth: {
      user: process.env.SMTP_USER,
      pass: process.env.SMTP_PASS,
    },
    pool: true,           // reuse TCP connections
    maxConnections: 3,
    maxMessages: 100,
  });

if (process.env.NODE_ENV !== "production") global.__mailer = mailer;
```

#### Next.js API route
```ts
import { NextRequest, NextResponse } from "next/server";
import { mailer }         from "@/utils/mailer";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { escapeHtml }     from "@/utils/escapeHtml";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";

export async function POST(req: NextRequest) {
  const raw = await req.text();
  if (raw.length > 50_000) return NextResponse.json({ error: "Request too large." }, { status: 413 });
  const body = JSON.parse(raw);

  const name    = sanitizeInput(body.name,    100);
  const email   = sanitizeInput(body.email,   254);
  const message = sanitizeInput(body.message, 5000);

  if (!name || !email) return NextResponse.json({ error: "Required fields missing." }, { status: 400 });
  const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
  if (!EMAIL_RE.test(email)) return NextResponse.json({ error: "Invalid email address." }, { status: 400 });

  const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim() ?? "unknown";
  if (!checkRateLimit(ip))                                 return NextResponse.json({ error: "Too many requests." }, { status: 429 });
  if (body.website)                                        return NextResponse.json({ success: true });
  if (!body._t || (Date.now() - Number(body._t)) < 3_000)  return NextResponse.json({ success: true });
  if (isDuplicate(email))                                  return NextResponse.json({ success: true });
  if (isSpam(message))                                     return NextResponse.json({ success: true });

  try {
    // Notify team
    await mailer.sendMail({
      from:    `"${process.env.SENDER_NAME}" <${process.env.SENDER_EMAIL}>`,
      to:      process.env.NOTIFY_EMAIL,
      replyTo: email,
      subject: `New submission from ${sanitizeInput(name, 100)}`,
      html: `
        <h2>New form submission</h2>
        <p><strong>Name:</strong> ${escapeHtml(name)}</p>
        <p><strong>Email:</strong> ${escapeHtml(email)}</p>
        <p><strong>Message:</strong><br/>${escapeHtml(message).replace(/\n/g, "<br/>")}</p>
      `,
    });

    // Confirm submitter
    await mailer.sendMail({
      from:    `"${process.env.SENDER_NAME}" <${process.env.SENDER_EMAIL}>`,
      to:      email,
      subject: "We received your message!",
      html:    `<p>Hi ${escapeHtml(name)},</p><p>Thanks for reaching out — we'll be in touch shortly.</p>`,
    });
  } catch (err) {
    console.error("[smtp] send failed:", err);
    return NextResponse.json({ error: "Could not send. Try again." }, { status: 502 });
  }

  return NextResponse.json({ success: true });
}
```

#### Env vars
```
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false            # true for 465, false for 587/STARTTLS
SMTP_USER=postmaster@yourdomain.com
SMTP_PASS=app-password-here
SENDER_NAME="Your Company"
SENDER_EMAIL=hello@yourdomain.com
NOTIFY_EMAIL=team@yourdomain.com
```

#### Setup notes
- **Google Workspace**: `smtp.gmail.com:587`, generate an App Password (Account → Security → 2-Step Verification → App passwords). Regular passwords are rejected.
- **Microsoft 365**: `smtp.office365.com:587`, STARTTLS, account must have SMTP AUTH enabled in Exchange admin.
- **Amazon SES**: `email-smtp.<region>.amazonaws.com:587`, generate SMTP credentials separately from IAM keys.
- **Connection pooling** (`pool: true`) is critical on long-running Node hosts. On serverless (Vercel/Netlify Functions) each invocation is a cold transport — that's fine, just slower per send.
- **Deliverability**: SPF + DKIM + DMARC on the sending domain are mandatory. Without them, expect inbox-spam-folder. See [deployment.md → DNS verification](deployment.md).

---

### Option 2 — Database persistence

Store every submission row in a database. Optionally also send a notification (combine with another option). Works on any stack where the API route can reach the DB.

#### Schema (shared shape — adapt to your engine)
```
submissions
  id          uuid primary key default gen_random_uuid()
  created_at  timestamptz not null default now()
  ip_hash     text not null
  name        text not null
  email       text not null
  message     text
  payload     jsonb not null     -- entire sanitized form for forward-compat
  status      text not null default 'received'
  index on created_at desc
  index on email
```

`ip_hash` is `sha256(ip + IP_SALT)` — store a salted hash, never the raw IP, to keep the table GDPR-friendly. See [compliance.md](compliance.md).

#### Option 2a — Prisma

`prisma/schema.prisma`:
```prisma
model Submission {
  id        String   @id @default(uuid())
  createdAt DateTime @default(now())
  ipHash    String
  name      String
  email     String
  message   String?
  payload   Json
  status    String   @default("received")

  @@index([createdAt(sort: Desc)])
  @@index([email])
}
```

`utils/db.ts`:
```ts
import { PrismaClient } from "@prisma/client";
declare global { var __prisma: PrismaClient | undefined; }
export const prisma = global.__prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== "production") global.__prisma = prisma;
```

API route delivery step (replaces the email send):
```ts
import { prisma } from "@/utils/db";
import { createHash } from "crypto";

const ipHash = createHash("sha256").update(ip + process.env.IP_SALT).digest("hex");

try {
  await prisma.submission.create({
    data: { ipHash, name, email, message, payload: body },
  });
} catch (err) {
  console.error("[db] write failed:", err);
  return NextResponse.json({ error: "Could not save. Try again." }, { status: 502 });
}
```

#### Option 2b — Supabase (no Prisma)

```ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,   // server-only key — never NEXT_PUBLIC_*
);

const { error } = await supabase.from("submissions").insert({
  ip_hash: ipHash, name, email, message, payload: body,
});
if (error) {
  console.error("[supabase] insert failed:", error);
  return NextResponse.json({ error: "Could not save. Try again." }, { status: 502 });
}
```

#### Option 2c — Raw Postgres (`pg`)
```ts
import { Pool } from "pg";
declare global { var __pgPool: Pool | undefined; }
const pool = global.__pgPool ?? new Pool({ connectionString: process.env.DATABASE_URL });
if (process.env.NODE_ENV !== "production") global.__pgPool = pool;

await pool.query(
  `INSERT INTO submissions (ip_hash, name, email, message, payload)
   VALUES ($1, $2, $3, $4, $5)`,
  [ipHash, name, email, message, body],
);
```

#### Env vars
```
DATABASE_URL=postgres://user:pass@host:5432/dbname
IP_SALT=<random 32-byte hex string>
# Supabase only:
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJ...
```

#### Setup notes
- **SQLite** works fine for self-hosted single-instance — set `DATABASE_URL="file:./data.db"`. Does **not** work on serverless or read-only filesystems.
- **Connection limits**: serverless functions can exhaust DB connection pools fast. Use a pooler (Supabase Pooler, Neon, PgBouncer, Prisma Data Proxy).
- **Retention**: add a scheduled job to purge old rows — see [compliance.md → Data retention](compliance.md).
- **Indexing**: the `(created_at desc)` index keeps the admin list view fast; the `email` index makes duplicate-checks cheap if you later move that guard to the DB.

---

### Option 3 — Slack / Discord / Teams webhook

No email at all — submissions ping a chat channel. Best for small teams, internal forms, or as a backup channel paired with Option 1 or 2.

#### Slack — incoming webhook (Block Kit)
```ts
async function notifySlack(name: string, email: string, message: string) {
  const res = await fetch(process.env.SLACK_WEBHOOK_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      text: `New submission from ${name}`,   // fallback for notifications
      blocks: [
        { type: "header", text: { type: "plain_text", text: "New form submission" } },
        {
          type: "section",
          fields: [
            { type: "mrkdwn", text: `*Name:*\n${name}` },
            { type: "mrkdwn", text: `*Email:*\n${email}` },
          ],
        },
        { type: "section", text: { type: "mrkdwn", text: `*Message:*\n${message || "_(none)_"}` } },
        { type: "context", elements: [{ type: "mrkdwn", text: `Received <!date^${Math.floor(Date.now()/1000)}^{date_short_pretty} at {time}|just now>` }] },
      ],
    }),
  });
  if (!res.ok) throw new Error(`Slack webhook ${res.status}`);
}
```

#### Discord — incoming webhook (embed)
```ts
async function notifyDiscord(name: string, email: string, message: string) {
  const res = await fetch(process.env.DISCORD_WEBHOOK_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      embeds: [{
        title: "New form submission",
        color: 0x5865F2,
        fields: [
          { name: "Name",  value: name,  inline: true },
          { name: "Email", value: email, inline: true },
          { name: "Message", value: message?.slice(0, 1024) || "(none)" },
        ],
        timestamp: new Date().toISOString(),
      }],
    }),
  });
  if (!res.ok) throw new Error(`Discord webhook ${res.status}`);
}
```

#### Microsoft Teams — incoming webhook (Adaptive Card)
```ts
async function notifyTeams(name: string, email: string, message: string) {
  const res = await fetch(process.env.TEAMS_WEBHOOK_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      type: "message",
      attachments: [{
        contentType: "application/vnd.microsoft.card.adaptive",
        content: {
          $schema: "http://adaptivecards.io/schemas/adaptive-card.json",
          type: "AdaptiveCard", version: "1.4",
          body: [
            { type: "TextBlock", size: "Medium", weight: "Bolder", text: "New form submission" },
            { type: "FactSet", facts: [{ title: "Name", value: name }, { title: "Email", value: email }] },
            { type: "TextBlock", text: message || "(no message)", wrap: true },
          ],
        },
      }],
    }),
  });
  if (!res.ok) throw new Error(`Teams webhook ${res.status}`);
}
```

#### Retry on 429
Webhook providers rate-limit. Wrap the call with a single retry honoring `Retry-After`:
```ts
async function postWithRetry(url: string, payload: unknown) {
  for (let attempt = 0; attempt < 2; attempt++) {
    const res = await fetch(url, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    });
    if (res.ok) return;
    if (res.status === 429 && attempt === 0) {
      const wait = Number(res.headers.get("retry-after") ?? 1) * 1000;
      await new Promise(r => setTimeout(r, wait));
      continue;
    }
    throw new Error(`webhook ${res.status}`);
  }
}
```

#### Env vars
```
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/T.../B.../...
# or
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/.../...
# or
TEAMS_WEBHOOK_URL=https://outlook.office.com/webhook/...
```

#### Setup notes
- **Slack**: App directory → "Incoming Webhooks" → add to channel → copy URL. Treat the URL as a secret; anyone with it can post.
- **Discord**: Server settings → Integrations → Webhooks → New Webhook → copy URL.
- **Teams**: Channel `…` → Connectors → Incoming Webhook → name + image → copy URL.
- **Always escape user content in chat?** Slack/Discord/Teams treat their own markup as literal in `text`/`value` fields — no HTML execution — but truncate `message` to keep cards readable (1024 chars for Discord field values).
- **Combine** with Option 2: write to DB first (durable record), then ping Slack (alert). If Slack fails, the submission is still safe.

---

### Option 4 — Generic outbound webhook (Zapier / Make / n8n / own backend)

Original stub — use this when the user already has a downstream system they want to POST to.

#### Client-side (works with any stack)
```ts
const res = await fetch(process.env.NEXT_PUBLIC_WEBHOOK_URL!, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(formData),
});
```

#### Minimal Next.js API route stub
```ts
import { NextRequest, NextResponse } from "next/server";
import { sanitizeInput }  from "@/utils/sanitizeInput";
import { checkRateLimit } from "@/utils/rateLimiter";
import { isDuplicate }    from "@/utils/duplicateGuard";
import { isSpam }         from "@/utils/spamFilter";

export async function POST(req: NextRequest) {
  const raw = await req.text();
  if (raw.length > 50_000) return NextResponse.json({ error: "Request too large." }, { status: 413 });
  const body = JSON.parse(raw);

  const name    = sanitizeInput(body.name,    100);
  const email   = sanitizeInput(body.email,   254);
  const message = sanitizeInput(body.message, 5000);

  if (!name || !email) return NextResponse.json({ error: "Required fields missing." }, { status: 400 });
  const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
  if (!EMAIL_RE.test(email)) return NextResponse.json({ error: "Invalid email address." }, { status: 400 });

  const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim() ?? "unknown";
  if (!checkRateLimit(ip))                                 return NextResponse.json({ error: "Too many requests." }, { status: 429 });
  if (body.website)                                        return NextResponse.json({ success: true });
  if (!body._t || (Date.now() - Number(body._t)) < 3_000)  return NextResponse.json({ success: true });
  if (isDuplicate(email))                                  return NextResponse.json({ success: true });
  if (isSpam(message))                                     return NextResponse.json({ success: true });

  try {
    const res = await fetch(process.env.WEBHOOK_URL!, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name, email, message, receivedAt: new Date().toISOString() }),
    });
    if (!res.ok) throw new Error(`webhook ${res.status}`);
  } catch (err) {
    console.error("[webhook] forward failed:", err);
    return NextResponse.json({ error: "Could not deliver. Try again." }, { status: 502 });
  }

  return NextResponse.json({ success: true });
}
```

---

## File Attachments

All five services support attaching uploaded files to the notification email. Use this pattern whenever a form has an `<input type="file">` field.

### Size limits
| Service | Max total attachment size |
|---------|--------------------------|
| Brevo | 10 MB |
| Resend | 40 MB |
| SendGrid | 30 MB |
| Mailgun | 25 MB |
| Postmark | 10 MB |

Enforce limits client-side first (see `accessibility-patterns.md → File Upload with Validation`) and server-side as a safeguard.

### Frontend change — use `FormData` instead of JSON

When a form includes a file field, submit as `multipart/form-data`:

```tsx
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  setLoading(true);
  try {
    const fd = new FormData();
    // Append all text fields
    Object.entries(form).forEach(([k, v]) => {
      if (typeof v === 'string') fd.append(k, v);
      if (Array.isArray(v))     fd.append(k, v.join(', '));
    });
    // Append file (fileRef is a React ref on the <input type="file">)
    if (fileRef.current?.files?.[0]) {
      fd.append('resume', fileRef.current.files[0]);
    }

    const res = await fetch('/api/contact', { method: 'POST', body: fd });
    // Note: do NOT set Content-Type header — browser sets multipart boundary automatically
    if (!res.ok) throw new Error((await res.json()).error ?? 'Something went wrong');
    setSubmitted(true);
  } catch (err: any) {
    setError(err.message);
  } finally {
    setLoading(false);
  }
};
```

### Server — read the file from `req.formData()`

```ts
export async function POST(req: NextRequest) {
  const fd    = await req.formData();
  const name  = fd.get('name')   as string;
  const email = fd.get('email')  as string;
  const file  = fd.get('resume') as File | null;

  // Server-side size guard (e.g. 5 MB)
  if (file && file.size > 5 * 1024 * 1024) {
    return NextResponse.json({ error: 'File exceeds 5 MB limit.' }, { status: 400 });
  }

  // Convert to base64 buffer for email services
  let attachmentBuffer: Buffer | null = null;
  let attachmentName = '';
  let attachmentType = '';
  if (file && file.size > 0) {
    attachmentBuffer = Buffer.from(await file.arrayBuffer());
    attachmentName   = file.name;
    attachmentType   = file.type;
  }

  // ... send email with attachment below
}
```

### Per-service attachment syntax

**Brevo**
```ts
await apiInstance.sendTransacEmail({
  // ...existing fields...
  attachment: attachmentBuffer ? [{
    name:    attachmentName,
    content: attachmentBuffer.toString('base64'),
  }] : undefined,
});
```

**Resend**
```ts
await resend.emails.send({
  // ...existing fields...
  attachments: attachmentBuffer ? [{
    filename: attachmentName,
    content:  attachmentBuffer,   // Resend accepts Buffer directly
  }] : undefined,
});
```

**SendGrid**
```ts
await sgMail.send({
  // ...existing fields...
  attachments: attachmentBuffer ? [{
    filename:    attachmentName,
    content:     attachmentBuffer.toString('base64'),
    type:        attachmentType,
    disposition: 'attachment',
  }] : undefined,
});
```

**Mailgun**
```ts
// Mailgun requires the raw buffer via `attachment`
await mg.messages.create(process.env.MAILGUN_DOMAIN!, {
  // ...existing fields...
  attachment: attachmentBuffer ? {
    filename: attachmentName,
    data:     attachmentBuffer,
  } : undefined,
});
```

**Postmark**
```ts
await client.sendEmail({
  // ...existing fields...
  Attachments: attachmentBuffer ? [{
    Name:        attachmentName,
    Content:     attachmentBuffer.toString('base64'),
    ContentType: attachmentType,
  }] : undefined,
});
```

Add hidden fields to capture lead attribution — populate client-side from URL params:

```tsx
// In component, on mount:
useEffect(() => {
  const params = new URLSearchParams(window.location.search);
  setForm(prev => ({
    ...prev,
    utm_source: params.get("utm_source") ?? "",
    utm_campaign: params.get("utm_campaign") ?? "",
    utm_medium: params.get("utm_medium") ?? "",
  }));
}, []);
```

Include these in the notification email table so the team can see which campaign drove the lead.
