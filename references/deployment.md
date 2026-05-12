# Deployment

Read this file **only when the user has tested the form locally and explicitly asks to deploy** ("deploy", "ship it", "push to prod", "go live"). Do not run any of this during initial form generation — dev and deploy are separate phases.

The Deploy phase is a guided walkthrough, not a single command. Go through it step by step and pause for the user to confirm each section before moving on.

---

## Pre-flight check (ask the user, do not assume)

Before touching anything production:

1. Does `npm run build` complete with zero TypeScript / build errors?
2. Did a real local submission land in the configured destination (team inbox / DB row / Slack message)?
3. Are all bot guards firing? (honeypot non-empty → silent 200; submit within 3s → silent 200)
4. Is `.env.local` in `.gitignore`? (`git check-ignore .env.local` should print the path)

If any answer is no, stop and fix locally first.

---

## Step 1 — Pick the target platform

Ask the user where the app will run, then jump to the matching subsection. The form's API route is the only piece that needs server runtime; the page itself is static.

| Platform | Runtime model | Env-var location |
|----------|---------------|------------------|
| Vercel | Serverless functions | Project → Settings → Environment Variables |
| Netlify | Serverless functions | Site → Site configuration → Environment variables |
| Cloudflare Pages | Edge / Workers | Project → Settings → Environment variables (Production) |
| Railway | Long-running container | Project → Variables tab |
| Fly.io | Long-running container | `fly secrets set KEY=value` |
| Render | Long-running container | Service → Environment |
| Docker / VPS (self-hosted) | Long-running container | `.env` file on host, mounted into container |

### Serverless vs long-running — what changes

| Concern | Serverless (Vercel/Netlify/CF) | Long-running (Railway/Fly/VPS) |
|---------|-------------------------------|--------------------------------|
| In-memory rate limiter | **Resets every cold start — unsafe.** Use Upstash. | Works as-is. |
| In-memory duplicate guard | Same — unsafe. Use Redis / DB. | Works as-is. |
| SMTP connection pooling | New transport per invocation (slower but fine) | Pooled, fast |
| File-based persistence | Filesystem is ephemeral / read-only | OK if disk is mounted |
| Cold start | First request after idle = 1–3s | None |

If the user picked a serverless target, **the rate limiter and duplicate guard must be upgraded to Upstash before going live**. See `security-patterns.md → Upstash Redis Rate Limiter`. This is not optional on serverless.

---

## Step 2 — Set environment variables on the platform

Mirror everything from `.env.local` into the platform's secret store. Walk through the list with the user. Every variable below that does not start with `NEXT_PUBLIC_` must be marked **secret / encrypted** on platforms that distinguish.

### Public vs server-only

| Visibility | Variables |
|------------|-----------|
| Server-only (secret) | `*_API_KEY`, `RECAPTCHA_SECRET_KEY`, `SMTP_PASS`, `DATABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SLACK_WEBHOOK_URL`, `DISCORD_WEBHOOK_URL`, `TEAMS_WEBHOOK_URL`, `WEBHOOK_URL`, `IP_SALT`, `SENDER_EMAIL`, `NOTIFY_EMAIL`, `ALLOWED_ORIGINS` |
| Browser-visible (`NEXT_PUBLIC_*`) | `NEXT_PUBLIC_RECAPTCHA_SITE_KEY` only |

**Hard rule:** if a value would compromise the system when leaked, it must **never** be prefixed `NEXT_PUBLIC_`. The Resend/Brevo/SendGrid API key and the Supabase service-role key are the most commonly misconfigured — double-check these.

### Platform notes

**Vercel** — set per environment (Production / Preview / Development). For Preview, use a separate sender domain or sandbox key so test submissions don't pollute the prod CRM list. Redeploy after adding vars; existing deployments do not pick them up.

**Netlify** — scope to "Same value for all deploy contexts" by default; override per branch only if needed. `netlify env:set KEY value` from CLI works too.

**Cloudflare Pages** — set under "Production" *and* "Preview" separately. CF Pages Functions run on Workers — Node-only packages (e.g. `nodemailer`) **will not run** on the default edge runtime. Either pick Resend/Brevo (HTTPS-based) or switch the route's runtime to `nodejs` if your framework supports it.

**Railway / Render / Fly** — same `KEY=value` block, no platform-specific gotchas. On Fly: `fly secrets set KEY=value` triggers a redeploy automatically.

**Docker / VPS** — keep production secrets in `/etc/<app>/env` owned `root:root`, mode `0600`, sourced via systemd `EnvironmentFile=`. Never bake secrets into the image.

---

## Step 3 — Verify sender domain (email paths only)

Skip this step entirely for DB-only or Slack/Discord/Teams-only setups.

Required for **any** outbound email, whether via Brevo / Resend / SendGrid / Mailgun / Postmark / your own SMTP. Without these records, mail lands in spam or is rejected outright by Gmail/Outlook (DMARC enforcement since 2024).

### Records to add at the DNS provider

| Record | What it does | Provided by |
|--------|--------------|-------------|
| **SPF** (TXT on root) | Authorizes sending IPs | Email service or your SMTP host |
| **DKIM** (CNAME or TXT on a subdomain) | Cryptographically signs outgoing mail | Email service dashboard |
| **DMARC** (TXT on `_dmarc.yourdomain.com`) | Tells receivers what to do with unauthenticated mail | You — start with `p=none` |

Minimal starting DMARC: `v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com`

### Verify the records resolve

Run before the first production send:
```bash
dig +short TXT yourdomain.com | grep spf
dig +short TXT default._domainkey.yourdomain.com    # selector varies by provider
dig +short TXT _dmarc.yourdomain.com
```

Or paste the domain into [https://mxtoolbox.com/SuperTool.aspx](https://mxtoolbox.com/SuperTool.aspx) and run SPF, DKIM, and DMARC lookups. All three must return a record before going live.

If the email service has its own "Verified" status indicator in the dashboard, wait for it to turn green — DNS can take up to 48h to propagate but usually completes in minutes.

---

## Step 4 — Add the origin check to the API route

The skill's Phase 5 contract includes an Origin check as step 0, but it is only meaningful once you know the production URL. Confirm `ALLOWED_ORIGINS` is set:

```
ALLOWED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com
```

For staging/preview deploys, add the preview URL as well — Vercel preview URLs change per deploy, so for those use a wildcard suffix check or skip the guard in non-production. The route already short-circuits with `403 Forbidden` if the `Origin` header is not in the allow-list (see `security-patterns.md → Origin guard`).

If the form is embedded cross-origin (iframed on a partner site), add that origin too.

---

## Step 5 — Production smoke test

Deploy, then run **two** tests from the user's actual prod URL.

### 5a — Real form submission
1. Open `https://yourdomain.com/contact` in a fresh browser tab.
2. Fill in name + email + message normally. Wait at least 4 seconds before clicking submit (the 3-second timer guard is on).
3. Confirm the success screen appears.
4. Confirm the destination received it:
   - Email path → team inbox + sender confirmation
   - DB path → row in `submissions` with `created_at` ≈ now
   - Slack/Discord/Teams → message in the channel

### 5b — Bot path returns silent 200
Use `curl` to send a request that should be silently dropped, and confirm no real delivery happens:
```bash
curl -i https://yourdomain.com/api/contact \
  -H "Content-Type: application/json" \
  -H "Origin: https://yourdomain.com" \
  -d '{"name":"Bot","email":"bot@test.com","message":"hi","website":"trap","_t":1}'
```
Expected: `200 {"success":true}` and **nothing** in the inbox / DB / channel. If anything arrives, the honeypot wiring is broken.

### 5c — Rate limit fires
```bash
for i in {1..10}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    https://yourdomain.com/api/contact \
    -H "Content-Type: application/json" \
    -H "Origin: https://yourdomain.com" \
    -d '{"name":"x","email":"x@y.com","message":"x","_t":'"$(($(date +%s)*1000-5000))"'}'
done
```
Expected: a mix of `200` and (after the 5th request from the same IP) `429`. If all return `200`, the limiter is not engaging — most likely because it's the in-memory variant on serverless. Switch to Upstash.

---

## Step 6 — Lighthouse + axe accessibility pass

From Chrome DevTools on the deployed URL:
1. **Lighthouse** → Categories: Accessibility → Generate report. Target ≥ 95.
2. **axe DevTools** extension → scan the contact page → resolve any "Serious" or "Critical" issues.

Common production-only failures:
- Color contrast on brand-colored submit button (looks fine on desktop, fails on the inspector)
- Missing `lang` on `<html>` (framework default, easy to add)
- Focus ring removed by global `outline: none` in a parent style sheet

---

## Step 7 — Monitoring (optional but recommended)

The API route already `console.error`s on real failures (email send 5xx, DB insert failure, webhook non-2xx). Surface those logs:

| Platform | Where to read |
|----------|---------------|
| Vercel | Project → Logs → filter by `/api/contact` |
| Netlify | Site → Logs → Functions |
| Cloudflare | Project → Functions → Real-time logs |
| Railway / Fly / Render | Service → Logs (live tail) |
| Docker | `docker logs -f <container>` or journald |

For higher signal, plug in one of: Sentry (free tier), Axiom, Logflare, Better Stack. Wire `console.error` automatically to whichever the user already uses — don't add a new tool unprompted.

Optional alert: if the `console.error` rate exceeds 1/min, page the team. Most form failures are deliverability issues that compound silently.

---

## Step 8 — Post-deploy hygiene

- Confirm `.env.local` is **not** in the latest commit (`git log --all --full-history -- .env.local` should be empty).
- Rotate any API key that was ever pasted into chat, Slack, or screen-shared.
- Note the deploy date — schedule a quarterly review of the DMARC report (it's the only way you'll learn about deliverability regressions before they cost leads).
- If using a DB, schedule the retention purge job — see [compliance.md](compliance.md).

---

## Quick recap — output to the user at the end of Deploy phase

```
✅ Deployed to: https://yourdomain.com
✅ Env vars set on: <platform>
✅ DNS verified: SPF / DKIM / DMARC all resolve
✅ Smoke test: real submission delivered + bot path silent + rate limit fires
✅ Lighthouse a11y: <score>

📅 Recurring:
   [ ] Review DMARC reports monthly
   [ ] Run retention purge weekly (if using DB)
   [ ] Rotate API keys yearly
```
