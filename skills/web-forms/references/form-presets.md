# Form Presets

Field definitions for each supported form type. Each field entry shows:
`name | type | label | placeholder | required | notes`

UI pattern reminder (from SKILL.md Phase 3):
- 4+ predefined options → **multi-select pill toggles**
- 2–3 mutually exclusive → **single-select pill row**
- Open/freeform → `<input>` or `<textarea>`

---

## Consulting Preset

**Business type:** consulting, advisory, professional services
**Default CTA:** "Book a Discovery Call"

### Required fields
| Name | Type | UI | Label | Options / Placeholder |
|------|------|----|-------|----------------------|
| name | text | input | Full Name | Jane Smith |
| email | email | input | Work Email | jane@company.com |
| company | text | input | Company | Acme Corp |

### Default fields (include unless user removes)
| Name | Type | UI | Label | Options |
|------|------|----|-------|---------|
| role | text | input | Your Role / Title | CEO, Head of Product… |
| service_interest | checkbox | **multi-select pills** | What can we help with? | Strategy · Operations · Finance · Marketing · Technology · People & Culture · Other |
| team_size | radio | **single-select pills** | Team Size | 1–10 · 11–50 · 51–200 · 200+ |
| timeline | radio | **single-select pills** | Timeline | ASAP · 1–3 months · 3–6 months · Exploring |
| budget | radio | **single-select pills** | Budget Range | <$10k · $10–50k · $50–150k · $150k+ |
| message | textarea | textarea | Anything else you'd like us to know? | — |

### Optional fields
| Name | Type | Label | Notes |
|------|------|-------|-------|
| phone | tel | Phone Number | Add if sales team does calls |
| referral | text / select | How did you hear about us? | Word of mouth / LinkedIn / Search / Event / Other |

---

## SaaS Preset

**Business type:** SaaS, software product, startup
**Default CTA:** "Request a Demo"

### Required fields
| Name | Type | UI | Label |
|------|------|----|-------|
| name | text | input | Full Name |
| email | email | input | Work Email |

### Default fields
| Name | Type | UI | Label | Options |
|------|------|----|-------|---------|
| company | text | input | Company | — |
| use_case | checkbox | **multi-select pills** | What are you looking to solve? | Automation · Analytics · Collaboration · Reporting · Integrations · Compliance · Other |
| team_size | radio | **single-select pills** | Team Size | Solo · 2–10 · 11–50 · 51–500 · 500+ |
| current_tools | text | input | Tools you currently use | Notion, Airtable, Salesforce… |
| timeline | radio | **single-select pills** | When are you looking to get started? | Right away · This quarter · Just exploring |
| message | textarea | textarea | Anything else? | — |

### Optional
| Name | Notes |
|------|-------|
| utm_source / utm_campaign | Hidden fields; auto-populate from URL params |
| phone | Only if SDR team does discovery calls |

---

## Agency Preset

**Business type:** agency, studio, creative, marketing
**Default CTA:** "Start a Project"

### Required fields
| Name | Type | UI | Label |
|------|------|----|-------|
| name | text | input | Your Name |
| email | email | input | Email Address |

### Default fields
| Name | Type | UI | Label | Options |
|------|------|----|-------|---------|
| company | text | input | Company / Brand | — |
| project_type | checkbox | **multi-select pills** | What kind of project? | Brand Identity · Web Design · Web Development · Campaign · Content · Social Media · Video · Other |
| budget | radio | **single-select pills** | Budget | <$5k · $5–15k · $15–50k · $50k+ |
| timeline | radio | **single-select pills** | Desired Timeline | ASAP · 1 month · 2–3 months · Flexible |
| message | textarea | textarea | Tell us about the project | Goals, context, references… |
| referral | select | select | How did you find us? | Google · Referral · Social · Portfolio · Other |

---

## Contact Preset (Generic)

**Purpose:** General website contact — any business type, questions, support
**Default CTA:** "Send Message"

### Required fields
| Name | Type | Label | Placeholder |
|------|------|-------|-------------|
| name | text | Full Name | Jane Smith |
| email | email | Email Address | jane@example.com |
| message | textarea | Message | How can we help? |

### Optional fields
| Name | Type | Label | Notes |
|------|------|-------|-------|
| phone | tel | Phone Number | Add if phone support offered |
| subject | text / select | Subject | Helps route messages |
| company | text | Company | For B2B context |
| preferred_contact | radio → **pills** | Preferred Contact | Email · Phone · Either |

### Layout
- Single column, mobile-first; message textarea 4–6 rows

---

## Lead Generation Form

**Purpose:** Capture potential customers — demo requests, quote requests, newsletter sign-ups, free trial.

### Required fields
| Name | Type | Label | Placeholder | Required |
|------|------|-------|-------------|----------|
| name | text | Full Name | Jane Smith | ✅ |
| email | email | Work Email | jane@company.com | ✅ |

### Optional fields (offer to user)
| Name | Type | Label | Notes |
|------|------|-------|-------|
| company | text | Company Name | Critical for B2B |
| role | text / select | Job Title | Helps qualify leads |
| team_size | select | Team Size | 1–10 / 11–50 / 51–200 / 200+ |
| phone | tel | Phone | Only if sales team needs it |
| interest | select / checkbox | I'm interested in… | Maps to products/services |
| message | textarea | Anything else? | Keep optional to reduce friction |
| utm_source | hidden | — | Auto-populate from URL params |
| utm_campaign | hidden | — | Auto-populate from URL params |

### Layout
- Keep it short — fewer fields = higher conversion
- Two-column layout works for name + email on desktop
- UTM hidden fields: always include for lead attribution

---

## Job Application Form

**Purpose:** Collect applications for open roles.

### Required fields
| Name | Type | Label | Placeholder | Required |
|------|------|-------|-------------|----------|
| first_name | text | First Name | Jane | ✅ |
| last_name | text | Last Name | Smith | ✅ |
| email | email | Email Address | jane@example.com | ✅ |
| phone | tel | Phone Number | +1 (555) 000-0000 | ✅ |
| role | text / select | Position Applying For | — | ✅ |
| resume | file | Resume / CV | .pdf, .doc, .docx (max 5MB) | ✅ |

### Optional fields (offer to user)
| Name | Type | Label | Notes |
|------|------|-------|-------|
| linkedin | url | LinkedIn Profile | Common for professional roles |
| portfolio | url | Portfolio / GitHub | For creative/dev roles |
| cover_letter | textarea | Cover Letter | 500–1000 word limit |
| start_date | date | Earliest Start Date | |
| location | text | Current Location | City, Country |
| work_auth | radio | Are you authorized to work in [country]? | Yes / No / Will require sponsorship |
| salary_expectation | text | Salary Expectation | |
| referral | text | How did you hear about us? | |

### Layout
- Group into sections: Personal Info → Application Details → Supporting Documents
- File upload: validate type (PDF/DOC) and size (≤5MB) client-side
- For multiple roles: use a `<select>` pre-populated with open positions

---

## Requirements Gathering Form

**Purpose:** Collect project scope, briefs, or onboarding information from clients.

### Required fields
| Name | Type | Label | Placeholder | Required |
|------|------|-------|-------------|----------|
| name | text | Your Name | Jane Smith | ✅ |
| email | email | Email Address | jane@company.com | ✅ |
| company | text | Company / Organisation | Acme Corp | ✅ |
| project_name | text | Project Name / Title | Website Redesign | ✅ |
| description | textarea | Project Description | Tell us about your project… | ✅ |
| goals | textarea | Goals & Success Criteria | What does success look like? | ✅ |

### Optional fields (offer to user)
| Name | Type | Label | Notes |
|------|------|-------|-------|
| budget | select | Estimated Budget | Range bands work better than free text |
| timeline | select | Desired Timeline | ASAP / 1 month / 3 months / Flexible |
| audience | textarea | Target Audience | Who will use this product? |
| competitors | textarea | Competitors / Inspiration | |
| existing_assets | checkbox | Do you have existing brand assets? | Logo / Style guide / Copy / None |
| attachments | file | Attachments (briefs, mockups) | Multiple files, 10MB max |
| priority | radio | What's the top priority? | Speed / Quality / Cost |
| decision_makers | text | Who else is involved in decisions? | |

### Layout
- Multi-step strongly recommended here (long form)
- Step 1: Contact Info → Step 2: Project Overview → Step 3: Details & Constraints

---

## Survey / Feedback Form

**Purpose:** Collect ratings, opinions, NPS scores, or structured feedback.

### Required fields (varies widely — these are defaults)
| Name | Type | Label | Notes |
|------|------|-------|-------|
| nps | range / radio | How likely are you to recommend us? (0–10) | NPS standard |
| overall_rating | radio / select | Overall satisfaction | 1–5 stars or Likert scale |
| feedback | textarea | Tell us more | Open-ended follow-up |

### Optional fields (offer to user)
| Name | Type | Label | Notes |
|------|------|-------|-------|
| name | text | Your Name | Make optional for anonymous surveys |
| email | email | Email (optional) | For follow-up |
| product_used | select | Which product/service? | |
| experience_area | checkbox | What did you experience? | Multiple choice |
| improvement | textarea | What could we do better? | |
| permission | checkbox | May we quote your feedback? | GDPR-friendly consent |

### Layout patterns:
- **NPS**: Large 0–10 button row with labels "Not likely" and "Extremely likely"
- **Star rating**: Use `<fieldset>` with radio inputs styled as stars
- **Likert scale**: Table layout on desktop, stacked on mobile
- Survey forms often benefit from one-question-at-a-time layout (reduces overwhelm)

---

## Security Fields (always include in every form)

These two fields are **invisible to users** and require no design changes.
They must appear in every generated form regardless of preset or business type.
Full copy-paste code in `references/security-patterns.md`.

### Honeypot field
A hidden `<input name="website">` placed just before the submit button.
Real users never see or touch it. Bots auto-fill it.
Server silently returns `200 { success: true }` when the field is non-empty — never a `400`.

```html
<!-- Place just before the submit button -->
<div aria-hidden="true" style="position:absolute;left:-9999px;height:0;overflow:hidden">
  <label for="website">Leave this blank</label>
  <input type="text" id="website" name="website" tabindex="-1" autocomplete="off" value="">
</div>
```

React equivalent: see `references/security-patterns.md → Honeypot`.
Add `website: ""` to the `FormData` type and `INITIAL` state.

### Form timer hidden field
Records the timestamp when the page loaded. Server rejects submissions that arrive in less than 3 seconds — faster than any human can read and fill a form.

```html
<!-- Place just before the submit button -->
<input type="hidden" id="_t" name="_t">
<script>document.getElementById('_t').value = Date.now();</script>
```

React equivalent: `const [formLoadTime] = useState(() => Date.now());` — include `_t: formLoadTime` in the JSON body on submit.

---

## Common Field Patterns

### Phone number
```html
<input type="tel" id="phone" name="phone"
  autocomplete="tel"
  pattern="[+]?[0-9\s\-\(\)]{7,15}"
  placeholder="+1 (555) 000-0000">
```

### File upload (resume/attachment)
```html
<input type="file" id="resume" name="resume"
  accept=".pdf,.doc,.docx"
  aria-describedby="resume-hint">
<p id="resume-hint">PDF, DOC or DOCX. Max 5MB.</p>
```
Add JS to validate `file.size` and `file.type` on change.

### Hidden UTM fields (auto-populate)
```html
<input type="hidden" id="utm_source" name="utm_source">
<input type="hidden" id="utm_campaign" name="utm_campaign">
<script>
  const params = new URLSearchParams(window.location.search);
  document.getElementById('utm_source').value = params.get('utm_source') || '';
  document.getElementById('utm_campaign').value = params.get('utm_campaign') || '';
</script>
```

### Honeypot (spam prevention)
See `references/security-patterns.md → Honeypot` for full HTML and React patterns.
Also see the **Security Fields** section above — the honeypot is mandatory in every form.
