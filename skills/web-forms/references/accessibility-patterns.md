# Accessibility & Advanced Validation Patterns

Reference for WCAG 2.1 AA compliant form patterns and complex field types.
Read this file when building file uploads, star ratings, conditional fields, or multi-step forms.

---

## Core Accessibility Rules (always apply)

### 1. Labels
```html
<!-- ✅ Correct: explicit label -->
<label for="email">Email Address</label>
<input type="email" id="email" name="email">

<!-- ✅ Correct: aria-label (use when visible label would be redundant) -->
<input type="search" aria-label="Search products">

<!-- ❌ Wrong: placeholder-only (disappears on focus) -->
<input type="email" placeholder="Email Address">
```

### 2. Required Fields
```html
<!-- Mark in HTML and visually -->
<label for="name">Full Name <span aria-hidden="true">*</span></label>
<input type="text" id="name" name="name" required aria-required="true">

<!-- Add a legend near the form top -->
<p><span aria-hidden="true">*</span> Required fields</p>
```

### 3. Error Messages
```html
<!-- Inline error — announced by screen readers -->
<label for="email">Email</label>
<input type="email" id="email" name="email"
  aria-describedby="email-error"
  aria-invalid="true">
<p id="email-error" role="alert" class="error">
  Please enter a valid email address.
</p>
```

JS to show/hide errors:
```js
function showError(inputId, message) {
  const input = document.getElementById(inputId);
  const error = document.getElementById(inputId + '-error');
  input.setAttribute('aria-invalid', 'true');
  error.textContent = message;
  error.hidden = false;
}

function clearError(inputId) {
  const input = document.getElementById(inputId);
  const error = document.getElementById(inputId + '-error');
  input.removeAttribute('aria-invalid');
  error.hidden = true;
}
```

### 4. Focus Management
```css
/* Never remove focus ring without replacement */
:focus-visible {
  outline: 3px solid #4A90E2;
  outline-offset: 2px;
}
```

On form submit with errors: move focus to first invalid field:
```js
const firstInvalid = form.querySelector('[aria-invalid="true"]');
if (firstInvalid) firstInvalid.focus();
```

### 5. Submit Button States
```html
<button type="submit" id="submit-btn">
  <span id="btn-text">Send Message</span>
  <span id="btn-loading" hidden aria-hidden="true">Sending…</span>
</button>
```
```js
function setSubmitting(isSubmitting) {
  const btn = document.getElementById('submit-btn');
  btn.disabled = isSubmitting;
  document.getElementById('btn-text').hidden = isSubmitting;
  document.getElementById('btn-loading').hidden = !isSubmitting;
  // Update aria for screen readers
  btn.setAttribute('aria-busy', isSubmitting);
}
```

---

## Complex Field Patterns

### Star Rating (Survey)
```html
<fieldset>
  <legend>Overall Rating</legend>
  <!-- Inputs before labels, CSS reversal for visual star effect -->
  <div class="star-rating" role="group">
    <input type="radio" id="star5" name="rating" value="5">
    <label for="star5" title="5 stars">★</label>
    <input type="radio" id="star4" name="rating" value="4">
    <label for="star4" title="4 stars">★</label>
    <input type="radio" id="star3" name="rating" value="3">
    <label for="star3" title="3 stars">★</label>
    <input type="radio" id="star2" name="rating" value="2">
    <label for="star2" title="2 stars">★</label>
    <input type="radio" id="star1" name="rating" value="1">
    <label for="star1" title="1 star">★</label>
  </div>
</fieldset>
```
Use CSS `direction: rtl` + `flex-direction: row-reverse` trick to highlight stars on hover.

### NPS Score (0–10)
```html
<fieldset>
  <legend>How likely are you to recommend us? (0 = Not likely, 10 = Extremely likely)</legend>
  <div class="nps-scale" role="group">
    <!-- Generate 0-10 radio buttons -->
    <!-- Each: <input type="radio" id="nps-N" name="nps" value="N"> -->
    <!-- Each: <label for="nps-N">N</label> -->
  </div>
  <div class="nps-labels" aria-hidden="true">
    <span>Not likely</span>
    <span>Extremely likely</span>
  </div>
</fieldset>
```

### File Upload with Validation
```html
<div class="file-upload">
  <label for="resume">Resume / CV</label>
  <input type="file" id="resume" name="resume"
    accept=".pdf,.doc,.docx"
    aria-describedby="resume-hint resume-error">
  <p id="resume-hint" class="hint">PDF, DOC or DOCX. Max 5MB.</p>
  <p id="resume-error" role="alert" hidden class="error"></p>
  <!-- Show selected filename -->
  <p id="resume-selected" aria-live="polite"></p>
</div>
```
```js
document.getElementById('resume').addEventListener('change', function() {
  const file = this.files[0];
  const errorEl = document.getElementById('resume-error');
  const selectedEl = document.getElementById('resume-selected');

  if (!file) return;

  const validTypes = ['application/pdf',
    'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document'];
  const maxSize = 5 * 1024 * 1024; // 5MB

  if (!validTypes.includes(file.type)) {
    errorEl.textContent = 'Please upload a PDF, DOC, or DOCX file.';
    errorEl.hidden = false;
    this.value = '';
    return;
  }
  if (file.size > maxSize) {
    errorEl.textContent = 'File must be smaller than 5MB.';
    errorEl.hidden = false;
    this.value = '';
    return;
  }

  errorEl.hidden = true;
  selectedEl.textContent = `Selected: ${file.name}`;
});
```

---

## Conditional & Progressive Patterns

Three patterns, each with HTML and React/Tailwind versions.

---

### Pattern 1 — Field Show/Hide

Reveal or hide 1–3 dependent fields based on a single answer.

**HTML version**
```html
<label for="contact-method">Preferred contact method</label>
<select id="contact-method" name="contact_method">
  <option value="email">Email</option>
  <option value="phone">Phone</option>
</select>

<!-- Hidden by default; do NOT mark required here — JS handles it -->
<div id="phone-group" hidden>
  <label for="phone">Phone number <span aria-hidden="true">*</span></label>
  <input type="tel" id="phone" name="phone" disabled>
  <p id="phone-error" role="alert" hidden class="error"></p>
</div>
```
```js
const trigger = document.getElementById('contact-method');
const group   = document.getElementById('phone-group');
const input   = document.getElementById('phone');

trigger.addEventListener('change', function () {
  const show = this.value === 'phone';
  group.hidden   = !show;
  input.disabled = !show;        // excluded from submission when hidden
  input.required = show;
  input.setAttribute('aria-required', String(show));
  if (show) input.focus();       // move focus to first revealed field
  else if (document.activeElement === input) trigger.focus(); // return focus on hide
});
```

**React/Tailwind version**
```tsx
const [contactMethod, setContactMethod] = useState<'email' | 'phone'>('email');

<div className="space-y-4">
  <div>
    <label htmlFor="contact-method" className="block font-medium mb-1">
      Preferred contact method
    </label>
    <select
      id="contact-method"
      value={contactMethod}
      onChange={e => setContactMethod(e.target.value as 'email' | 'phone')}
      className="w-full border rounded px-3 py-2"
    >
      <option value="email">Email</option>
      <option value="phone">Phone</option>
    </select>
  </div>

  {contactMethod === 'phone' && (
    <div>
      <label htmlFor="phone" className="block font-medium mb-1">
        Phone number <span aria-hidden="true">*</span>
      </label>
      <input
        id="phone"
        type="tel"
        required
        autoFocus          // focus moves here on reveal
        className="w-full border rounded px-3 py-2"
      />
    </div>
  )}
</div>
```

---

### Pattern 2 — Section Skip

An entire `<fieldset>` / `<section>` is irrelevant for one answer path. Use when a key qualifier ("Are you an individual or a business?") gates a whole block of fields.

**HTML version**
```html
<fieldset id="type-selector">
  <legend>Account type <span aria-hidden="true">*</span></legend>
  <label><input type="radio" name="account_type" value="individual" checked> Individual</label>
  <label><input type="radio" name="account_type" value="business"> Business</label>
</fieldset>

<!-- Business-only section — hidden initially -->
<section id="business-section" hidden aria-labelledby="biz-heading">
  <h3 id="biz-heading">Business details</h3>
  <label for="company">Company name <span aria-hidden="true">*</span></label>
  <input type="text" id="company" name="company" disabled>
  <label for="vat">VAT number</label>
  <input type="text" id="vat" name="vat" disabled>
</section>
```
```js
const radios  = document.querySelectorAll('input[name="account_type"]');
const section = document.getElementById('business-section');
// All inputs inside the section (for bulk enable/disable)
const inputs  = section.querySelectorAll('input, select, textarea');

radios.forEach(radio => {
  radio.addEventListener('change', function () {
    const show = this.value === 'business';
    section.hidden = !show;
    inputs.forEach(el => {
      el.disabled = !show;
      // restore required only on fields that need it
      if (el.dataset.required) el.required = show;
    });
    if (show) section.querySelector('input, select, textarea').focus();
  });
});
```
Mark fields that need `required` with `data-required="true"` in HTML so the JS above can restore it.

**React/Tailwind version**
```tsx
const [accountType, setAccountType] = useState<'individual' | 'business'>('individual');

{/* Qualifier pill row */}
<fieldset>
  <legend className="font-medium mb-2">Account type *</legend>
  <div className="flex gap-3">
    {(['individual', 'business'] as const).map(type => (
      <button
        key={type} type="button"
        onClick={() => setAccountType(type)}
        aria-pressed={accountType === type}
        className={`px-4 py-2 rounded-full border text-sm font-medium transition-colors
          ${accountType === type
            ? 'bg-[#ACCENT] text-white border-[#ACCENT]'
            : 'bg-white text-gray-700 border-gray-300 hover:border-[#ACCENT]'}`}
      >
        {type.charAt(0).toUpperCase() + type.slice(1)}
      </button>
    ))}
  </div>
</fieldset>

{/* Business-only section — conditionally rendered */}
{accountType === 'business' && (
  <section aria-labelledby="biz-heading" className="space-y-4 border-t pt-4 mt-4">
    <h3 id="biz-heading" className="font-semibold">Business details</h3>
    <div>
      <label htmlFor="company" className="block font-medium mb-1">Company name *</label>
      <input id="company" type="text" required autoFocus
        className="w-full border rounded px-3 py-2" />
    </div>
    <div>
      <label htmlFor="vat" className="block font-medium mb-1">VAT number</label>
      <input id="vat" type="text" className="w-full border rounded px-3 py-2" />
    </div>
  </section>
)}
```

---

### Pattern 3 — Adaptive Multi-Step

Steps 2+ change content or order based on a qualifier answered in step 1.

**Step routing logic (React)**
```tsx
type AccountType = 'individual' | 'business';

// Define step sequences per path
const STEP_SEQUENCES: Record<AccountType, string[]> = {
  individual: ['contact', 'preferences', 'review'],
  business:   ['contact', 'company',     'billing', 'review'],
};

const [accountType, setAccountType] = useState<AccountType>('individual');
const [stepIndex, setStepIndex]     = useState(0);

const steps    = STEP_SEQUENCES[accountType];
const current  = steps[stepIndex];
const total    = steps.length;
const isLast   = stepIndex === total - 1;

function next() {
  if (validateCurrentStep()) setStepIndex(i => i + 1);
}
function back() {
  setStepIndex(i => i - 1);
}

// Progress bar
<div role="progressbar" aria-valuenow={stepIndex + 1}
     aria-valuemin={1} aria-valuemax={total}
     aria-label={`Step ${stepIndex + 1} of ${total}: ${current}`}
     className="text-sm text-gray-500 mb-6">
  Step {stepIndex + 1} of {total}
</div>

{/* Render only the current step */}
{current === 'contact'     && <ContactStep     onNext={next} />}
{current === 'preferences' && <PreferencesStep onNext={next} onBack={back} />}
{current === 'company'     && <CompanyStep     onNext={next} onBack={back} />}
{current === 'billing'     && <BillingStep     onNext={next} onBack={back} />}
{current === 'review'      && <ReviewStep      onBack={back} onSubmit={handleSubmit} />}
```

**Focus management between steps**
```tsx
const stepHeadingRef = useRef<HTMLHeadingElement>(null);

useEffect(() => {
  // Move focus to step heading whenever step changes
  stepHeadingRef.current?.focus();
}, [stepIndex]);

// In each step component's heading:
<h2 tabIndex={-1} ref={stepHeadingRef} className="text-xl font-bold mb-4">
  {stepTitle}
</h2>
```

**Preserving state across steps**
Keep a single top-level state object; pass a partial updater down to each step:
```tsx
const [formData, setFormData] = useState<FormData>(INITIAL);
const patch = (fields: Partial<FormData>) =>
  setFormData(prev => ({ ...prev, ...fields }));

// Pass to step: <CompanyStep data={formData} patch={patch} ... />
```
This ensures Back never loses entered values and the final Review step can display a complete summary.

### Checkbox Group (Multi-select)
```html
<fieldset>
  <legend>Which services are you interested in? <span aria-hidden="true">*</span></legend>
  <div class="checkbox-group">
    <div>
      <input type="checkbox" id="svc-web" name="services" value="web">
      <label for="svc-web">Web Design</label>
    </div>
    <div>
      <input type="checkbox" id="svc-seo" name="services" value="seo">
      <label for="svc-seo">SEO</label>
    </div>
    <div>
      <input type="checkbox" id="svc-brand" name="services" value="brand">
      <label for="svc-brand">Branding</label>
    </div>
  </div>
  <p id="services-error" role="alert" hidden class="error">
    Please select at least one service.
  </p>
</fieldset>
```

---

## Multi-Step Form Pattern (fixed steps)

For adaptive steps that change based on earlier answers, see **Conditional & Progressive Patterns → Pattern 3** above.

This section covers the simpler fixed-step case (same steps for everyone).

HTML structure:
```html
<form id="multi-form" novalidate>
  <!-- Progress indicator -->
  <div class="progress" role="progressbar"
    aria-valuenow="1" aria-valuemin="1" aria-valuemax="3"
    aria-label="Step 1 of 3">
    Step <span id="current-step">1</span> of 3
  </div>

  <!-- Steps (only one visible at a time) -->
  <section id="step-1" class="step" aria-labelledby="step1-heading">
    <h2 id="step1-heading">Your Details</h2>
    <!-- fields -->
  </section>
  <section id="step-2" class="step" hidden aria-labelledby="step2-heading">
    <h2 id="step2-heading">Project Info</h2>
    <!-- fields -->
  </section>

  <!-- Navigation -->
  <div class="step-nav">
    <button type="button" id="prev-btn" hidden>Back</button>
    <button type="button" id="next-btn">Next</button>
    <button type="submit" id="submit-btn" hidden>Submit</button>
  </div>
</form>
```

JS: Validate current step fields before advancing. On step change, update `aria-valuenow` and move focus to the new step's `<h2>` heading (set `tabIndex="-1"` on it). Preserve all entered values in a single state object so Back never resets fields.

---

## Conversational / Typeform-style Pattern

One question at a time, full-screen slides, keyboard navigation, animated transitions.

### Question definition shape
```tsx
type Question = {
  id: string;
  label: string;            // visible question text
  type: 'text' | 'email' | 'tel' | 'textarea' | 'select' | 'radio' | 'pills';
  options?: string[];       // for select / radio / pills
  required?: boolean;
  placeholder?: string;
};

const QUESTIONS: Question[] = [
  { id: 'name',    label: 'What's your name?',           type: 'text',  required: true },
  { id: 'email',   label: 'And your email?',             type: 'email', required: true },
  { id: 'service', label: 'What are you looking for?',   type: 'pills',
    options: ['Strategy', 'Design', 'Development', 'Marketing'], required: true },
  { id: 'message', label: 'Tell us more.',               type: 'textarea' },
];
```

### State & navigation
```tsx
const [index, setIndex]       = useState(0);
const [answers, setAnswers]   = useState<Record<string, string | string[]>>({});
const [direction, setDir]     = useState<'forward' | 'back'>('forward');
const [animKey, setAnimKey]   = useState(0);   // bump to re-trigger CSS animation
const inputRef = useRef<HTMLInputElement | HTMLTextAreaElement | null>(null);

const current  = QUESTIONS[index];
const total    = QUESTIONS.length;
const isLast   = index === total - 1;

function validateCurrent(): boolean {
  if (!current.required) return true;
  const val = answers[current.id];
  if (Array.isArray(val)) return val.length > 0;
  return !!val?.trim();
}

function advance() {
  if (!validateCurrent()) { /* show inline error */ return; }
  setDir('forward');
  setAnimKey(k => k + 1);
  setIndex(i => i + 1);
}

function retreat() {
  setDir('back');
  setAnimKey(k => k + 1);
  setIndex(i => i - 1);
}

// Move focus to input after slide transition
useEffect(() => {
  const t = setTimeout(() => inputRef.current?.focus(), 320); // match CSS duration
  return () => clearTimeout(t);
}, [index]);
```

### Keyboard handler
```tsx
function handleKeyDown(e: React.KeyboardEvent) {
  if (e.key === 'Enter' && current.type !== 'textarea') {
    e.preventDefault();
    isLast ? handleSubmit() : advance();
  }
  if (e.key === 'ArrowRight' || e.key === 'ArrowDown') advance();
  if (e.key === 'ArrowLeft'  || e.key === 'ArrowUp')   retreat();
}
```
Attach `onKeyDown={handleKeyDown}` to the outer wrapper `<div>`.

### CSS slide animation
```css
@keyframes slideInRight  { from { opacity: 0; transform: translateX(60px);  } to { opacity: 1; transform: translateX(0); } }
@keyframes slideInLeft   { from { opacity: 0; transform: translateX(-60px); } to { opacity: 1; transform: translateX(0); } }
@keyframes slideOutLeft  { from { opacity: 1; transform: translateX(0); }  to { opacity: 0; transform: translateX(-60px);  } }
@keyframes slideOutRight { from { opacity: 1; transform: translateX(0); }  to { opacity: 0; transform: translateX(60px); } }

.slide-in-forward  { animation: slideInRight  0.3s ease forwards; }
.slide-in-back     { animation: slideInLeft   0.3s ease forwards; }
```

Apply to the question wrapper:
```tsx
<section
  key={animKey}
  className={direction === 'forward' ? 'slide-in-forward' : 'slide-in-back'}
  aria-labelledby={`q-${current.id}`}
>
```

Or with Framer Motion:
```tsx
import { AnimatePresence, motion } from 'framer-motion';

const variants = {
  enter:  (dir: string) => ({ x: dir === 'forward' ?  60 : -60, opacity: 0 }),
  center:                  ({ x: 0,  opacity: 1 }),
  exit:   (dir: string) => ({ x: dir === 'forward' ? -60 :  60, opacity: 0 }),
};

<AnimatePresence custom={direction} mode="wait">
  <motion.section
    key={index}
    custom={direction}
    variants={variants}
    initial="enter" animate="center" exit="exit"
    transition={{ duration: 0.28, ease: 'easeInOut' }}
  >
    {/* question content */}
  </motion.section>
</AnimatePresence>
```

### Progress bar + counter
```tsx
<div className="flex items-center gap-3 mb-8">
  <div
    role="progressbar"
    aria-valuenow={index + 1}
    aria-valuemin={1}
    aria-valuemax={total}
    aria-label={`Question ${index + 1} of ${total}`}
    className="flex-1 h-1 bg-gray-200 rounded-full overflow-hidden"
  >
    <div
      className="h-full bg-[#ACCENT] transition-all duration-300"
      style={{ width: `${((index + 1) / total) * 100}%` }}
    />
  </div>
  <span className="text-sm text-gray-500 shrink-0">{index + 1} / {total}</span>
</div>
```

### Full question renderer
```tsx
function QuestionInput({ q, value, onChange }: {
  q: Question;
  value: string | string[];
  onChange: (val: string | string[]) => void;
}) {
  const strVal = Array.isArray(value) ? '' : (value ?? '');

  if (q.type === 'textarea') return (
    <textarea
      ref={inputRef as React.RefObject<HTMLTextAreaElement>}
      id={q.id} rows={4} value={strVal} required={q.required}
      placeholder={q.placeholder ?? 'Your answer…'}
      onChange={e => onChange(e.target.value)}
      className="w-full bg-transparent border-b-2 border-gray-300 focus:border-[#ACCENT]
                 outline-none resize-none text-xl py-2 transition-colors"
    />
  );

  if (q.type === 'pills') {
    const selected = Array.isArray(value) ? value : [];
    return (
      <div role="group" aria-label={q.label} className="flex flex-wrap gap-3 mt-4">
        {q.options?.map(opt => (
          <button key={opt} type="button"
            onClick={() => onChange(
              selected.includes(opt) ? selected.filter(s => s !== opt) : [...selected, opt]
            )}
            aria-pressed={selected.includes(opt)}
            className={`px-5 py-2 rounded-full border-2 text-base font-medium transition-all
              ${selected.includes(opt)
                ? 'bg-[#ACCENT] text-white border-[#ACCENT]'
                : 'bg-white text-gray-700 border-gray-300 hover:border-[#ACCENT]'}`}
          >
            {opt}
          </button>
        ))}
      </div>
    );
  }

  return (
    <input
      ref={inputRef as React.RefObject<HTMLInputElement>}
      id={q.id} type={q.type} value={strVal} required={q.required}
      placeholder={q.placeholder ?? 'Type your answer…'}
      onChange={e => onChange(e.target.value)}
      className="w-full bg-transparent border-b-2 border-gray-300 focus:border-[#ACCENT]
                 outline-none text-2xl py-2 transition-colors placeholder:text-gray-300"
    />
  );
}
```

### Full layout shell
```tsx
// Full-screen centered layout
<div
  className="min-h-screen flex flex-col justify-center items-center bg-white px-6"
  onKeyDown={handleKeyDown}
>
  {/* Progress */}
  <div className="w-full max-w-xl mb-8">{/* progress bar */}</div>

  {/* Animated question */}
  <div className="w-full max-w-xl">
    <AnimatePresence custom={direction} mode="wait">
      <motion.section key={index} /* variants, transitions */>
        <label id={`q-${current.id}`} htmlFor={current.id}
          className="block text-3xl font-semibold mb-6">
          {current.label}
          {current.required && <span aria-hidden="true" className="text-[#ACCENT] ml-1">*</span>}
        </label>
        <QuestionInput q={current} value={answers[current.id] ?? ''}
          onChange={val => setAnswers(a => ({ ...a, [current.id]: val }))} />
        {error && <p role="alert" className="text-red-500 mt-2 text-sm">{error}</p>}
      </motion.section>
    </AnimatePresence>
  </div>

  {/* Navigation */}
  <div className="w-full max-w-xl flex justify-between mt-10">
    {index > 0 && (
      <button type="button" onClick={retreat}
        className="text-gray-400 hover:text-gray-700 text-sm underline">
        ← Back
      </button>
    )}
    <div className="ml-auto">
      {isLast
        ? <button type="button" onClick={handleSubmit} disabled={loading}
            className="px-8 py-3 bg-[#ACCENT] text-white rounded-full font-medium">
            {loading ? 'Submitting…' : 'Submit'}
          </button>
        : <button type="button" onClick={advance}
            className="px-8 py-3 bg-[#ACCENT] text-white rounded-full font-medium">
            OK ↵
          </button>
      }
    </div>
  </div>

  <p className="text-xs text-gray-400 mt-4">Press Enter ↵ to continue</p>
</div>
```

### Accessibility notes for Typeform-style
- Every question section needs `aria-labelledby` pointing to its `<label id>`
- Hidden questions use both `hidden` attribute AND `aria-hidden="true"` — the animation library may render them in the DOM briefly during transition
- The progress bar `aria-valuenow` must update with each step change
- "Press Enter to continue" hint: mark as `aria-hidden="true"` (it's a visual cue; keyboard users discover it naturally)
- After submit, replace the whole widget with a `role="status"` success message and move focus to it

---

| Requirement | Technique |
|-------------|-----------|
| Perceivable | Labels on all inputs, errors not color-only |
| Operable | Tab order, visible focus, no keyboard traps |
| Understandable | Clear labels, error messages describe fix |
| Robust | Semantic HTML, ARIA only to enhance (not replace) |

Test with: keyboard-only navigation, VoiceOver (Mac), NVDA (Windows), or axe DevTools browser extension.

---

## UI/UX heuristics for form polish

Used by the audit mode (SKILL.md Phase A) as the P2 rubric, and by Phase 4 of generation
as a final pass over the produced component. These are heuristics, not WCAG rules — they
turn a correct form into a *good* form.

### Visual hierarchy
- One clear H1 above the form (the form's purpose, not the page's nav title)
- Labels: consistent weight (medium/600), one size, never all-caps unless brand-systemic
- Input text **≥ 16px** on mobile — anything smaller triggers iOS zoom-on-focus
- Helper text smaller than label, muted gray, sits **between** label and input or directly below input — never floating to the right

### Spacing rhythm
- 16–24px vertical gap between fields
- 6–8px between label and its input
- 12–16px between input and inline error/helper
- Submit button sits **24–32px** below the last field, never glued to it

### Error UX
- Inline, directly below the offending field — never only in a banner
- Red text **plus** an icon or "Error:" prefix (color-only fails WCAG 1.4.1)
- Error persists until the user corrects the field, not until next keystroke
- On submit-with-errors: scroll the first invalid field into view AND focus it

### Mobile
- Tap targets **≥ 44×44px** (Apple HIG) — applies to checkboxes, radio pills, submit
- Inputs full-width on viewports < 640px
- Set `inputmode` (`email`, `tel`, `numeric`, `decimal`) and `autocomplete` (`email`, `name`, `tel`, `organization`, `street-address`, etc.) on every applicable field — saves the user 5–15 seconds and reduces typos
- `enterkeyhint="next"` on intermediate fields, `enterkeyhint="send"` on the last one
- For phone, email, URL/links, and date of birth, follow the per-field snippets in [field-validation.md](field-validation.md) — each ships the correct `type` + `inputmode` + `autocomplete` combo plus on-blur validation and 16px+ font (avoids iOS zoom-on-focus)

### Brand application
- **One** accent color: primary CTA background, focus ring, active pill toggle, progress bar
- Neutral grays for all chrome (borders, labels, helper text, disabled states)
- Hover/focus states must shift by at least ~10% luminance — not just an opacity change
- Avoid two competing accents; secondary actions are ghost buttons or text links

### Microcopy
- CTA: verb + noun ("Send message", "Book a call", "Request quote") — never just "Submit"
- Placeholders are **examples**, not labels (`e.g. acme.com`, not `Company`)
- Success state copy mirrors the CTA's promise ("We'll be in touch within 1 business day"), not generic "Thanks!"
- Error messages name the fix, not the violation ("Enter a valid email like name@company.com", not "Invalid format")

### Motion
- Subtle 150–250ms ease-out on focus, hover, and conditional reveals
- Stagger only in Typeform-style mode (one question at a time); never on a single-page form
- Wrap all transitions in `@media (prefers-reduced-motion: reduce) { * { transition: none !important } }` — required for vestibular safety

### Audit shortcut

When auditing an existing form, check these heuristics in this order: hierarchy → spacing → error UX → mobile attrs → brand color usage → microcopy → motion. The first three account for ~80% of "this form feels off" P2 findings.

---

## Polish Layer (P2 opt-in)

These are **not** included in default generation. Offer them as P2 wins in
Phase A audits, or apply when the user explicitly asks to "make the form
feel premium". Each is independent — pick the ones that fit the audience.

### 1. Floating labels
Label sits inside the input at rest, shrinks above on focus or when the field has a value. Clearer than placeholder-only (placeholder disappears the moment users start typing) and saves vertical space — great for long forms on mobile.

```tsx
<div className="relative">
  <input
    id="email" name="email" type="email" placeholder=" "
    className="peer w-full min-h-[44px] text-base px-3 pt-5 pb-1 border rounded
               focus:outline-none focus-visible:ring-2 focus-visible:ring-primary"
  />
  <label
    htmlFor="email"
    className="absolute left-3 top-3.5 text-gray-500 transition-all
               peer-focus:top-1 peer-focus:text-xs peer-focus:text-primary
               peer-[:not(:placeholder-shown)]:top-1 peer-[:not(:placeholder-shown)]:text-xs"
  >
    Email
  </label>
</div>
```

Rules: keep `placeholder=" "` (single space) so the `:not(:placeholder-shown)` selector works; never make the floating label the *only* label — keep `htmlFor` / `id` for screen readers.

### 2. Character counter on textareas
Show `0 / 500` under long-form fields. Turn red and `aria-invalid="true"` past the cap.

```tsx
const MAX = 500;
<p className={`text-xs mt-1 text-right ${message.length > MAX ? "text-red-600" : "text-gray-500"}`}
   aria-live="polite">
  {message.length} / {MAX}
</p>
```

`aria-live="polite"` makes the counter accessible without spamming the screen reader.

### 3. Button label morph
The submit button's text changes through the lifecycle instead of disappearing. Read by screen readers via `aria-live` on the button text.

```tsx
<button type="submit" disabled={loading || submitted} aria-busy={loading}
        className="min-h-[44px] px-6 rounded bg-primary text-white">
  <span aria-live="polite">
    {submitted ? "Sent ✓" : loading ? "Sending…" : "Send message"}
  </span>
</button>
```

Pair with a brief 1-second hold on "Sent ✓" before the success state replaces the form — feels confirmed, not abrupt.

### 4. Error summary at the top (long forms only — 8+ fields)
Render a list of errors at the top of the form after a failed submit, each linking to the offending field. Mirrors GOV.UK pattern.

```tsx
{errorList.length > 0 && (
  <div role="alert" tabIndex={-1} ref={summaryRef} className="border-2 border-red-600 p-4 mb-6">
    <h2 className="font-bold mb-2">There is a problem</h2>
    <ul className="list-disc pl-5">
      {errorList.map(({ id, msg }) => (
        <li key={id}><a href={`#${id}`} className="text-red-700 underline">{msg}</a></li>
      ))}
    </ul>
  </div>
)}
```

On submit failure, `summaryRef.current?.focus()` moves screen readers and keyboards straight to the summary; the anchor links then take them to each field. Replaces (does not duplicate) the existing "focus first invalid field" rule when present.

### 5. Dark-mode pass
If Phase 2.5 detected `dark:` variants anywhere in the project, every form surface gets a matching dark variant. Audit any existing form for missing dark variants on inputs, labels, borders, placeholder text, error text, and the success state.

```tsx
<input className="bg-white text-gray-900 border-gray-300 placeholder-gray-400
                  dark:bg-gray-900 dark:text-gray-100 dark:border-gray-700 dark:placeholder-gray-500" />
```

Test against `prefers-color-scheme: dark` *and* whatever class/data-attribute toggle the project uses.

### 6. Inline help text (under-label, not placeholder)
For fields where the format matters (DOB, phone with country code, password rules), put a short `id="*-help"` paragraph **above the input** and reference it from `aria-describedby`. Don't bury format rules in the placeholder — it vanishes the moment users focus.

### 7. Optimistic input states
When validation passes on blur, render a subtle ✓ inside the field's right padding. Don't combine with a green border — too loud. Keep the ring color reserved for focus.

```tsx
<div className="relative">
  <input className="pr-9 …" />
  {valid && !error && (
    <span aria-hidden="true" className="absolute right-3 top-1/2 -translate-y-1/2 text-green-600">✓</span>
  )}
</div>
```

### 8. Smart enter-key behavior
On multi-field forms, `Enter` in any text input submits — that's the default and usually fine. On Typeform-style and multi-step forms, intercept `Enter` to advance the step instead; use `enterkeyhint="next"` on intermediate fields and `enterkeyhint="send"` on the last one.

---

When the user agrees to the polish layer, apply items independently — don't
treat it as all-or-nothing. Items 1, 3, and 7 give the biggest "feels
premium" lift per line of code.

---

## Pill toggles (multi-select)

Use for 4+ predefined options. Replace `#ACCENT` with the brand color
from the interview (or the detected design token from Phase 4).

```tsx
const OPTIONS = ['Strategy', 'Design', 'Development', 'Marketing'];
const [selected, setSelected] = useState<string[]>([]);

const toggle = (item: string) =>
  setSelected(prev =>
    prev.includes(item) ? prev.filter(i => i !== item) : [...prev, item]
  );

<div role="group" aria-labelledby="services-label">
  <p id="services-label" className="font-medium mb-2">Services *</p>
  <div className="flex flex-wrap gap-2">
    {OPTIONS.map(opt => (
      <button
        key={opt} type="button"
        onClick={() => toggle(opt)}
        aria-pressed={selected.includes(opt)}
        className={`px-4 py-2 rounded-full border text-sm font-medium transition-colors
          ${selected.includes(opt)
            ? 'bg-[#ACCENT] text-white border-[#ACCENT]'
            : 'bg-white text-gray-700 border-gray-300 hover:border-[#ACCENT]'}`}
      >
        {opt}
      </button>
    ))}
  </div>
</div>
```

For single-select (2–3 mutually exclusive options), use the same pattern
with a single `selected` string and `aria-pressed` true on the chosen pill.

---

## Component shell

The standard React shell every generated form starts from. Replace the
typed fields and the success-state copy with values from the interview.
For redirect-on-success (Phase 2 Q11 = thank-you page), call
`useRouter().push("/thank-you")` in the try block instead of
`setSubmitted(true)`.

```tsx
"use client";
import { useState } from "react";

type FormData = { /* typed fields per preset */ };
const INITIAL: FormData = { /* defaults */ };

export default function ContactForm() {
  const [form, setForm]       = useState<FormData>(INITIAL);
  const [loading, setLoading] = useState(false);
  const [error, setError]     = useState<string | null>(null);
  const [submitted, setSubmitted] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true); setError(null);
    try {
      const res = await fetch("/api/contact", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(form),
      });
      if (!res.ok) throw new Error((await res.json()).error ?? "Something went wrong");
      setSubmitted(true);
    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  if (submitted) return (
    <div role="status" aria-live="polite" className="text-center py-16">
      <h2 className="text-2xl font-bold mb-2">You're all set!</h2>
      <p className="text-gray-600">/* CTA-specific copy from interview */</p>
    </div>
  );

  return (
    <form onSubmit={handleSubmit} noValidate>
      {error && <p role="alert" className="text-red-600 mb-4">{error}</p>}
      {/* fields */}
      <button type="submit" disabled={loading} aria-busy={loading}>
        {loading ? "Sending…" : "/* CTA label from interview */"}
      </button>
    </form>
  );
}
```
