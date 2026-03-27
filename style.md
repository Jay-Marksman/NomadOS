
***

# NomadOS Style Guide

This document defines the visual language for NomadOS so that documentation, the setup wizard, dashboards, and boot screens feel like one cohesive product.

***

## 1. Brand tone

NomadOS should feel calm, private, space‑themed, and trustworthy:

- Calm and low‑contrast dark surfaces, no harsh pure blacks.  
- Clean, modern typography with good readability at low DPI.  
- Sparse but meaningful animation; nothing flashy or distracting.  
- Minimal text and options on critical paths (boot, setup wizard).

***

## 2. Typography

### 2.1 Primary font stack

Use this stack everywhere possible (web UI, docs, dashboards):

```css
font-family: Inter, system-ui, -apple-system, BlinkMacSystemFont,
             "Segoe UI", Roboto, "Noto Sans", Arial, sans-serif;
```

- Inter: main typeface for UI and documentation.  
- Noto Sans: preferred fallback for languages Inter does not cover.

### 2.2 Weights and usage

- 700 – Headings (H1, important section titles).  
- 600 – Subheadings, button labels, key UI labels.  
- 400 – Body text, descriptions, helper text.  

Avoid extra‑thin or ultra‑light weights.

### 2.3 Sizes and spacing (web/UI)

Baseline recommendations:

- H1: 28–32 px, 700 weight, line-height 1.1–1.2.  
- H2: 22–24 px, 600 weight, line-height 1.2.  
- H3: 18–20 px, 600 weight, line-height 1.2–1.3.  
- Body: 15–17 px, 400 weight, line-height 1.4–1.6.  
- Small/help text: 13–14 px, 400 weight, line-height 1.4–1.6.

For long content (docs), prefer 16–17 px body size and keep line length around 70–90 characters.

***

## 3. Color system

### 3.1 Core palette (tokens)

Define these as design tokens in code:

```css
--nomad-bg:        #02030A;
--nomad-bg-deep:   #050814;
--nomad-surface:   #0B1020;

--nomad-text:      #E4E7F5;
--nomad-text-muted:#9BA4C6;

--nomad-primary:   #7FB5FF;  /* logo blue */
--nomad-primary-soft: #4E8AE6;
--nomad-accent:    #9A7DFF;  /* secondary accent */

--nomad-border-subtle: #20263A;
--nomad-border-strong: #36405A;

--nomad-danger:    #FF4B5C;
--nomad-success:   #3AD29F;
--nomad-warning:   #FFC857;
```

### 3.2 Usage guidelines

- Backgrounds  
  - App background: `--nomad-bg`.  
  - Panels, cards: `--nomad-surface`.  
  - Boot and splash screens: gradient from `--nomad-bg` to `--nomad-bg-deep`.

- Text  
  - Primary content: `--nomad-text`.  
  - Secondary labels, metadata: `--nomad-text-muted`.  
  - Avoid pure white text except on critical alerts.

- Actions  
  - Primary buttons: background `--nomad-primary-soft`, text `--nomad-bg`.  
  - Hover/active states: slightly lighten or darken the background by 6–10%.  
  - Links: `--nomad-primary`, underline on hover.

- Borders and dividers  
  - Use `--nomad-border-subtle` for card outlines and separators.  
  - Use `--nomad-border-strong` only for highly interactive elements (focused inputs, emphasized panels).

- Feedback states  
  - Success: accent with `--nomad-success`, never as a large background.  
  - Warning: accent with `--nomad-warning`, keep surrounding background dark to avoid glare.  
  - Danger: `--nomad-danger` for destructive actions (text or subtle outlines, not full backgrounds on large areas).

***

## 4. Layout and components

### 4.1 Spacing scale

Use a simple 4‑point spacing scale:

```text
4, 8, 12, 16, 24, 32, 48 px
```

- 8–12 px between related controls.  
- 16–24 px between major sections.  
- 24–32 px padding inside cards and main layout gutters.

### 4.2 Corners and elevation

- Border radius: 8 px for cards, buttons, and inputs.  
- Elevation:
  - Flat surfaces: no shadow.  
  - Raised cards: small soft shadow, e.g. `0 8px 24px rgba(0, 0, 0, 0.35)`.

### 4.3 Buttons

- Primary button:  
  - Background `--nomad-primary-soft`, text `--nomad-bg`.  
  - Hover: slightly lighter background.  
- Secondary button:  
  - Transparent or `--nomad-surface` background, border `--nomad-border-strong`, text `--nomad-text`.  
- Destructive button:  
  - Background `--nomad-danger`, text `--nomad-bg`.

Buttons should use 600 weight text and all‑caps only for very short labels (e.g., “OK”, “SAVE”). Prefer Title Case for longer labels.

***

## 5. Imagery and iconography

- Background imagery  
  - Space/stars motif like `splash.jpg`.  
  - Stars should be small and slightly soft; avoid dense clusters behind text.  
  - No pure‑white highlights; keep brightest stars below full white.

- Logo usage  
  - Primary lockup: circular “N” mark with “NomadOS” text, as in `splash.jpg`.  
  - For cramped spaces (favicon, system tray icon, Plymouth logo), use the circular mark only.  

- Icons  
  - Prefer simple, outlined or duotone icons that work at 16–24 px.  
  - Icon color: `--nomad-text-muted` by default; `--nomad-primary` for active states.

***

## 6. Boot and setup wizard alignment

To keep the feeling consistent from power‑on to logged‑in:

- Use the same palette and fonts in the setup wizard as in the boot branding.  
- Setup wizard background: solid `--nomad-bg` with a soft vignette or very faint stars; never reuse the full splash image behind text.  
- Key wizard headings and progress steps should use the same blue accent `--nomad-primary`.  
- Wizard CTA buttons (Next, Finish) should be primary buttons with `--nomad-primary-soft`.

***

## 7. Content tone for documentation and UI copy

- Voice: friendly, concise, technically competent.  
- Avoid jargon where possible; when unavoidable, give a one‑line explanation.  
- Use second person (“you”) and present tense.  
- Prefer short sentences and clear actions (“Click Continue to format your external drive.”).

***

## 8. Code snippets

Example CSS variables block for quick reuse:

```css
:root {
  --nomad-bg: #02030A;
  --nomad-bg-deep: #050814;
  --nomad-surface: #0B1020;

  --nomad-text: #E4E7F5;
  --nomad-text-muted: #9BA4C6;

  --nomad-primary: #7FB5FF;
  --nomad-primary-soft: #4E8AE6;
  --nomad-accent: #9A7DFF;

  --nomad-border-subtle: #20263A;
  --nomad-border-strong: #36405A;

  --nomad-danger: #FF4B5C;
  --nomad-success: #3AD29F;
  --nomad-warning: #FFC857;
}
```

***

