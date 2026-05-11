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

const apiInstance = new Brevo.TransactionalEmailsApi();
apiInstance.setApiKey(Brevo.TransactionalEmailsApiApiKeys.apiKey, process.env.BREVO_API_KEY!);

const contactsApi = new Brevo.ContactsApi();
contactsApi.setApiKey(Brevo.ContactsApiApiKeys.apiKey, process.env.BREVO_API_KEY!);

export async function POST(req: NextRequest) {
  const body = await req.json();
  const { name, email, ...rest } = body;

  // 1. Server-side validation
  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }

  // 2. Build submission table for notification email
  const rows = [
    ["Name", name], ["Email", email],
    ...Object.entries(rest).map(([k, v]) => [
      k.replace(/_/g, " ").replace(/\b\w/g, c => c.toUpperCase()),
      Array.isArray(v) ? v.join(", ") : String(v),
    ]),
  ];
  const tableRows = rows
    .map(([k, v]) => `<tr><td style="padding:6px 12px;font-weight:600">${k}</td><td style="padding:6px 12px">${v}</td></tr>`)
    .join("");

  // 3. Notify team
  await apiInstance.sendTransacEmail({
    sender: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    to: [{ email: process.env.NOTIFY_EMAIL! }],
    replyTo: { email, name },
    subject: `New enquiry from ${name}`,
    htmlContent: `
      <h2 style="font-family:sans-serif">New Contact Form Submission</h2>
      <table style="font-family:sans-serif;border-collapse:collapse">${tableRows}</table>
    `,
  });

  // 4. Confirm submitter
  await apiInstance.sendTransacEmail({
    sender: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    to: [{ email, name }],
    subject: "We received your message!",
    htmlContent: `
      <p style="font-family:sans-serif">Hi ${name},</p>
      <p style="font-family:sans-serif">Thanks for reaching out — we'll be in touch shortly.</p>
      <p style="font-family:sans-serif">— The Team</p>
    `,
  });

  // 5. Sync contact to Brevo CRM list (optional)
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

const resend = new Resend(process.env.RESEND_API_KEY);

export async function POST(req: NextRequest) {
  const body = await req.json();
  const { name, email, ...rest } = body;

  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }

  const rows = [
    ["Name", name], ["Email", email],
    ...Object.entries(rest).map(([k, v]) => [
      k.replace(/_/g, " ").replace(/\b\w/g, c => c.toUpperCase()),
      Array.isArray(v) ? v.join(", ") : String(v),
    ]),
  ];
  const tableHtml = rows
    .map(([k, v]) => `<tr><td style="padding:6px 12px;font-weight:600">${k}</td><td style="padding:6px 12px">${v}</td></tr>`)
    .join("");

  // Notify team
  await resend.emails.send({
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [process.env.NOTIFY_EMAIL!],
    reply_to: email,
    subject: `New enquiry from ${name}`,
    html: `<h2>New Form Submission</h2><table style="border-collapse:collapse">${tableHtml}</table>`,
  });

  // Confirm submitter
  await resend.emails.send({
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [email],
    subject: "We received your message!",
    html: `<p>Hi ${name},</p><p>Thanks for reaching out — we'll be in touch shortly.</p><p>— The Team</p>`,
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

sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

export async function POST(req: NextRequest) {
  const body = await req.json();
  const { name, email, ...rest } = body;

  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }

  const rows = Object.entries({ name, email, ...rest })
    .map(([k, v]) => `<tr><td><strong>${k}</strong></td><td>${Array.isArray(v) ? v.join(", ") : v}</td></tr>`)
    .join("");

  await sgMail.send({
    to: process.env.NOTIFY_EMAIL!,
    from: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    replyTo: email,
    subject: `New enquiry from ${name}`,
    html: `<table>${rows}</table>`,
  });

  await sgMail.send({
    to: email,
    from: { name: process.env.SENDER_NAME!, email: process.env.SENDER_EMAIL! },
    subject: "We received your message!",
    html: `<p>Hi ${name}, thanks for reaching out — we'll be in touch shortly.</p>`,
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

const mg = new Mailgun(FormData).client({
  username: "api",
  key: process.env.MAILGUN_API_KEY!,
  url: process.env.MAILGUN_REGION === "eu"
    ? "https://api.eu.mailgun.net"
    : "https://api.mailgun.net",
});

export async function POST(req: NextRequest) {
  const body = await req.json();
  const { name, email, ...rest } = body;

  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }

  const text = Object.entries({ name, email, ...rest })
    .map(([k, v]) => `${k}: ${Array.isArray(v) ? v.join(", ") : v}`)
    .join("\n");

  await mg.messages.create(process.env.MAILGUN_DOMAIN!, {
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [process.env.NOTIFY_EMAIL!],
    "h:Reply-To": email,
    subject: `New enquiry from ${name}`,
    text,
  });

  await mg.messages.create(process.env.MAILGUN_DOMAIN!, {
    from: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    to: [email],
    subject: "We received your message!",
    text: `Hi ${name},\n\nThanks for reaching out — we'll be in touch shortly.\n\n— The Team`,
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

const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN!);

export async function POST(req: NextRequest) {
  const body = await req.json();
  const { name, email, ...rest } = body;

  if (!name || !email) {
    return NextResponse.json({ error: "Name and email are required." }, { status: 400 });
  }

  const rows = Object.entries({ name, email, ...rest })
    .map(([k, v]) => `<tr><td><strong>${k}</strong></td><td>${Array.isArray(v) ? v.join(", ") : v}</td></tr>`)
    .join("");

  await client.sendEmail({
    From: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    To: process.env.NOTIFY_EMAIL!,
    ReplyTo: email,
    Subject: `New enquiry from ${name}`,
    HtmlBody: `<table>${rows}</table>`,
    MessageStream: "outbound",
  });

  await client.sendEmail({
    From: `${process.env.SENDER_NAME} <${process.env.SENDER_EMAIL}>`,
    To: email,
    Subject: "We received your message!",
    HtmlBody: `<p>Hi ${name},</p><p>Thanks for reaching out — we'll be in touch shortly.</p>`,
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

## No Email / Webhook Only

Use when the user wants to POST to their own backend, a Zapier webhook, Make (Integromat), etc.

### Client-side (works with any stack)
```ts
const res = await fetch(process.env.NEXT_PUBLIC_WEBHOOK_URL!, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(formData),
});
```

### Minimal Next.js API route stub
```ts
export async function POST(req: NextRequest) {
  const body = await req.json();
  const { name, email } = body;
  if (!name || !email) {
    return NextResponse.json({ error: "Required fields missing." }, { status: 400 });
  }
  // TODO: save to DB, forward to webhook, etc.
  console.log("Form submission:", body);
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
