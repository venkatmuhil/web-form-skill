# Design Context — Detect & Reuse the Project's Design System

Read this file in **Phase 2.5** (before code generation) and at the start of
**Phase A** (audit). The goal: when the project already has a design system,
the generated form should feel native — same tokens, same primitives — not
bolted on. When it doesn't, fall back cleanly to the Phase 1 Q5 brand colors.

The probe is **read-only and bounded** (≤ 7 files inspected). Skip the whole
phase only if the project is a standalone HTML file with no build step.

---

## Detection order (stop at the first match per category)

### A. Explicit design doc

| Path (any of) | What to read |
|---------------|--------------|
| `design.md`, `DESIGN.md`, `docs/design.md`, `docs/design-system.md` | Treat as authoritative. Pull: brand colors, type scale, spacing, radius, motion rules, accessibility notes, voice/microcopy guidance. Quote the doc when proposing component shape. |

If found, **show the user one line** before generating:
> "I read your `design.md` — I'll use the [X] accent, [Y] radius, and the existing `<Button>` component."

### B. Token sources (in priority order)

1. **`tailwind.config.{js,ts,mjs,cjs}`** — `theme.colors.*`, `theme.extend.colors.*`,
   `borderRadius`, `fontFamily`, `spacing`. If `colors.primary` / `colors.accent`
   exists, use `bg-primary` / `bg-accent` / `ring-primary` instead of arbitrary
   `bg-[#HEX]` values. Same for `rounded-DEFAULT` over `rounded-md` if customised.
2. **`tokens.json`, `design-tokens.json`, `tokens/*.json`** — Style Dictionary
   format. Map `color.primary.500` → CSS custom property the project already
   exposes (e.g. `--color-primary-500`); use `var(--color-primary-500)`.
3. **`globals.css`, `app/globals.css`, `styles/globals.css`** — search for
   `--color-*`, `--bg-*`, `--accent-*`, `--radius-*` CSS custom properties at
   `:root`. Use them via `var(--name)` instead of inlining hex.
4. **`shadcn/ui` detected** (`components/ui/button.tsx` exists) — see C below.

If two sources disagree (rare), prefer Tailwind config — it's the build-time truth.

### C. Component primitives (in priority order)

Reuse what's there instead of generating fresh markup:

| Library signal | Reuse |
|----------------|-------|
| `components/ui/{button,input,label,textarea,checkbox,select,form}.tsx` (shadcn) | `<Button>`, `<Input>`, `<Label>`, `<Textarea>`, `<Checkbox>` — import from `@/components/ui/*` |
| `@chakra-ui/react` in `package.json` | `<Button>`, `<Input>`, `<FormControl>`, `<FormLabel>`, `<FormErrorMessage>` |
| `@mui/material` | `<TextField>`, `<Button>`, `<FormHelperText>` |
| `react-aria-components` / `@radix-ui/react-*` | Use existing primitives if a wrapper exists in `components/`; otherwise fall through |
| Custom in-repo: `components/Form*.tsx`, `components/Input.tsx` | Read the file once, match its prop API |

**Rule:** if a primitive exists, **never** hand-roll a new one for this form.
The accessibility, focus-ring, and theming logic is already in the primitive —
duplicating it forks the design system.

### D. Storybook (lowest priority, optional)

If `.storybook/` exists, the canonical visual examples are in `*.stories.{ts,tsx,mdx}`.
Don't parse these — just note in the plan that the user can validate the
generated form against the Button/Input stories.

---

## What to extract (cheat sheet)

| Token | Looks like | Use it for |
|-------|-----------|-----------|
| Primary / accent color | `--color-primary`, `colors.primary.DEFAULT`, brand hex in `design.md` | Submit button bg, focus ring, active pill, progress bar |
| Neutrals | `colors.gray.*`, `--text-muted`, `--border` | Labels, borders, helper text, disabled |
| Border radius | `--radius`, `borderRadius.DEFAULT` | Inputs, buttons, cards — pick one and apply consistently |
| Font family | `fontFamily.sans` | Form body |
| Spacing scale | `spacing.*` | Gap between fields (typical: 4 or 6 units = 16–24 px) |
| Motion | `transition`, `animation` keyframes, `prefers-reduced-motion` | Reveal/hide for conditional sections |
| Dark mode | `dark:*` Tailwind variants, `data-theme="dark"`, `[data-theme=dark]` CSS | Apply matching `dark:` variants on every field |

---

## Apply to the generator (changes to the Phase 4 component shell)

When tokens / primitives are detected:

1. **Substitute tokens for inline hex.** `bg-[#FF6B6B]` → `bg-primary`. Focus
   ring `ring-[#FF6B6B]` → `ring-primary` or `ring-[var(--ring)]`.
2. **Import primitives instead of generating `<input>` / `<button>`.**
   ```tsx
   // Without detection
   <input className="w-full min-h-[44px] text-base px-3 py-2 border rounded …" />

   // With shadcn detected
   import { Input } from "@/components/ui/input";
   <Input className="…" />
   ```
3. **Use the project's Label component** if found — keeps `htmlFor` / `id`
   wiring consistent with the rest of the app.
4. **Keep field-validation.md snippets verbatim where they're not visual** —
   `onChange`/`onBlur` handlers, error state, `aria-*` attributes are
   logic, not design. Only the JSX wrappers change.
5. **Dark-mode pass** if `dark:` variants exist anywhere in the project: add
   `dark:bg-gray-900 dark:border-gray-700 dark:text-gray-100 dark:placeholder-gray-500`
   (or the matched token equivalents) to every field surface.

---

## When nothing is detected

Fall back cleanly:

- Use the Phase 1 Q5 brand color hex directly (current behavior — `bg-[#HEX]`).
- Use a sensible default radius (`rounded-md`) and the existing
  `min-h-[44px] text-base` mobile baseline.
- Don't invent a token file. The user didn't ask for a design system.

---

## Output line (always show after the probe)

After the probe runs, show **one line** before the Phase 1 confirmation:

```
🎨 Design context: [tailwind.config.ts: primary=#FF6B6B, radius=lg] +
   [shadcn/ui: Button, Input, Label, Textarea] — generated form will use these.
```

Or, if nothing was found:

```
🎨 Design context: none detected — will use brand color from Q5 with default radius / spacing.
```

This is the only visible artifact of Phase 2.5 — the user sees what was found
and can correct it ("actually use `--accent` not `--primary`") before any
generation happens.

---

## Phase A integration

In Audit mode, run the same probe. Add findings to the report when the
existing form **disagrees with the project's design system**:

| Finding | Severity |
|---------|----------|
| Form uses inline hex `bg-[#xxx]` but `tailwind.config.ts` defines `colors.primary` | P2 |
| Form hand-rolls `<input>` but `components/ui/input.tsx` exists | P2 |
| Form has no `dark:` variants but rest of project ships dark mode | P2 |
| Form ignores token in `design.md` (e.g. wrong radius) | P2 |

These are P2 (UI/UX polish) — not P0/P1 — so the user can decline them
without compromising security or accessibility.
