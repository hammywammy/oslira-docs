# Oslira Design System Technical Specification

**A professional B2B SaaS design system combining Stripe's generous spacing philosophy with Linear's efficient precision for data-driven platforms.**

---

## Typography System

### Primary Font Family

**Inter** serves as the primary typeface across all interfaces. This font was specifically designed for UI/screen readability with a tall x-height optimized for small sizes and data-heavy dashboards. Inter provides a variable font with weights 100-900 and is open-source.

```css
font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
```

### Type Scale Specifications

The typography system uses a modular scale anchored to a 16px base size with carefully calibrated line heights and letter spacing for optimal readability in dashboard contexts.

**H1 (Page Title)**
- Size: **48px** (3rem)
- Line height: **56px** (1.167)
- Font weight: **700**
- Letter spacing: **-0.02em**

**H2 (Section Header)**
- Size: **32px** (2rem)
- Line height: **40px** (1.25)
- Font weight: **600**
- Letter spacing: **-0.01em**

**H3 (Subsection)**
- Size: **24px** (1.5rem)
- Line height: **32px** (1.333)
- Font weight: **600**
- Letter spacing: **0**

**H4 (Card/Component Header)**
- Size: **20px** (1.25rem)
- Line height: **28px** (1.4)
- Font weight: **600**
- Letter spacing: **0**

**Body Large (Lead Text)**
- Size: **18px** (1.125rem)
- Line height: **28px** (1.556)
- Font weight: **400**
- Letter spacing: **0**

**Body (Default Paragraph)**
- Size: **16px** (1rem)
- Line height: **24px** (1.5)
- Font weight: **400**
- Letter spacing: **0**

**Body Small (Secondary Text)**
- Size: **14px** (0.875rem)
- Line height: **20px** (1.429)
- Font weight: **400**
- Letter spacing: **0**

**Caption (Helper Text, Metadata)**
- Size: **13px** (0.8125rem)
- Line height: **18px** (1.385)
- Font weight: **400**
- Letter spacing: **0.01em**

**Small (Fine Print, Timestamps)**
- Size: **12px** (0.75rem)
- Line height: **16px** (1.333)
- Font weight: **400**
- Letter spacing: **0.01em**

**Label (UI Text, Buttons, Form Labels)**
- Size: **14px** (0.875rem)
- Line height: **20px** (1.429)
- Font weight: **500**
- Letter spacing: **0**

### Font Weight Usage

The system employs four specific weights for all typography needs:

- **400 (Regular):** Default body text and paragraphs
- **500 (Medium):** UI labels, button text, emphasized inline text
- **600 (Semibold):** H3-H4 headings, form labels, navigation items
- **700 (Bold):** H1-H2 headings, primary call-to-action buttons

### Paragraph Spacing Standards

- **Between paragraphs:** 16px (1rem)
- **After headings:** 12px (0.75rem) before body content
- **Before section headings:** 32px (2rem) for clear visual separation
- **List item spacing:** 8px (0.5rem) between individual items

---

## Spacing & Layout System

### Base Spacing Unit

The system employs an **8px base unit** with 4px half-steps for precision adjustments. This provides visual distinction while maintaining reasonable token counts and supports 1.5x scaling across different displays.

### Complete Spacing Scale

```
Token    Value     Rem        Use Case
─────────────────────────────────────────────────────
0        0px       0          Reset/none
xs       4px       0.25rem    Icon padding, tight spacing
sm       8px       0.5rem     Compact element spacing
md       16px      1rem       DEFAULT base spacing unit
lg       24px      1.5rem     Section spacing, card padding
xl       32px      2rem       Large component padding
2xl      48px      3rem       Major section spacing
3xl      64px      4rem       Page section breaks
4xl      80px      5rem       Hero section spacing
```

### Container Padding Standards

**Page Container (Horizontal)**
- Mobile (<768px): **16px**
- Tablet (768-1024px): **24px**
- Desktop (>1024px): **32px**

**Component Internal Padding**
- Compact cards/list items: **12px**
- Default cards/modals: **24px**
- Feature cards: **32px**
- Hero sections: **48px**

### Section Spacing Rules

**Vertical Rhythm Between Major Sections**
- Tight layout: **48px** (3rem)
- Default layout: **64px** (4rem) — **RECOMMENDED**
- Generous layout: **80px** (5rem)
- Hero to content transition: **96px** (6rem)

**Within Section Content**
- Heading to content: **16px**
- Content blocks (paragraphs/cards): **24px**
- Form field spacing: **16px**
- Button groups: **12px**

### Grid System Specifications

**Column Grid**
- Columns: **12** (divisible by 2, 3, 4, 6)
- Gutter width: **24px** (generous, aligned with Stripe philosophy)
- Margin (page edges): Responsive based on container padding

**Container Max-Widths**
- Narrow (text-heavy, forms): **640px**
- Default (standard content): **1140px** — **PRIMARY**
- Wide (dashboards, data tables): **1280px**
- Full-width (media-rich): **1440px**

**Responsive Breakpoints**
- sm: **640px**
- md: **768px**
- lg: **1024px**
- xl: **1280px**
- 2xl: **1536px**

---

## Border Radius System

The border radius system provides subtle, professional rounding that enhances the modern aesthetic without compromising the data-driven focus.

**Complete Border Radius Scale**
- **None:** 0px (tables, strict data displays)
- **Small (sm):** 4px (badges, tags, tight spaces)
- **Medium (md):** 6px (buttons, inputs) — **PRIMARY**
- **Large (lg):** 8px (cards, panels)
- **Extra Large (xl):** 12px (modals, large cards)
- **2XL:** 16px (hero cards, feature sections)
- **Full/Circle:** 9999px (pills, avatars)

### Usage Guidelines

- **Buttons:** 6px (medium) — Universal standard across all button types
- **Form inputs:** 6px (medium) — Consistency with buttons
- **Small cards:** 8px (large)
- **Large cards/panels:** 12px (xl)
- **Modals/dialogs:** 12px (xl)
- **Pills/tags:** 9999px (full)
- **Avatars:** 50% or 9999px (circle)
- **Data tables:** 0px (none) — Maintains professional precision
- **Images within cards:** 4px (small)

---

## Elevation & Shadow System

Shadows create depth through subtle, layered effects using very low opacity to maintain the clean, professional aesthetic. All shadows use black (rgba(0,0,0,...)) as the base color for consistency.

### Shadow Specifications

**Level 0 - None**
```css
box-shadow: none;
```
**Usage:** Flat elements, disabled states, within elevated containers

**Level 1 - Raised (Subtle)**
```css
box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04), 0 1px 3px rgba(0, 0, 0, 0.02);
```
**Usage:** Default buttons, subtle cards, slightly elevated content
- Y-offset: 1px, Blur: 2-3px, Opacity: 0.02-0.04

**Level 2 - Elevated (Standard)**
```css
box-shadow: 0 2px 4px rgba(0, 0, 0, 0.06), 0 1px 2px rgba(0, 0, 0, 0.04);
```
**Usage:** Cards, panels, content blocks, navigation bars — **PRIMARY**
- Y-offset: 1-2px, Blur: 2-4px, Opacity: 0.04-0.06

**Level 3 - Overlay (Dropdown)**
```css
box-shadow: 0 4px 8px rgba(0, 0, 0, 0.08), 0 2px 4px rgba(0, 0, 0, 0.06);
```
**Usage:** Dropdowns, popovers, tooltips, floating action buttons
- Y-offset: 2-4px, Blur: 4-8px, Opacity: 0.06-0.08

**Level 4 - Modal (Maximum)**
```css
box-shadow: 0 8px 16px rgba(0, 0, 0, 0.12), 0 4px 8px rgba(0, 0, 0, 0.08);
```
**Usage:** Modals, dialogs, major overlay panels
- Y-offset: 4-8px, Blur: 8-16px, Opacity: 0.08-0.12

### Shadow Principles

Each shadow level uses **two layered shadows**: a closer, sharper shadow that defines the edge, and a further, softer shadow that creates depth. Shadows follow a progressive doubling pattern (1→2→4→8px) for consistent elevation perception. Maximum opacity never exceeds 0.12 to maintain the subtle, professional aesthetic.

---

## Component State Patterns

### Button States

**Default (Enabled)**
- Full color saturation with defined background color
- Clear visual hierarchy through color and weight

**Hover State**
- Color change: Background darkens by **10%**
- Transition timing: **150ms**
- Cursor: pointer
- No transforms or scale effects (maintains solid, predictable feel)

**Active/Pressed State**
- Transform: **scale(0.98)**
- Transition timing: **0ms** (instant feedback)
- Shadow: Reduced to Level 0 (none) for depression effect

**Focus State**
- Ring width: **2px** solid outline
- Ring offset: **2px** from element edge
- Ring color: Primary brand color
- Implementation: `box-shadow: 0 0 0 2px var(--color-primary)`
- Transition timing: **100ms** (must appear quickly for keyboard navigation)
- Applies to all interactive elements (buttons, inputs, links)

**Disabled State**
- Opacity: **0.5**
- Cursor: not-allowed
- Color: Desaturated (maintains brand hue but reduced saturation)
- ARIA: aria-disabled="true"
- Tab focusable but prevents activation

### Card States

**Hover (Interactive Cards)**
- Transform: **translateY(-2px)** for subtle lift effect
- Shadow: Elevate from Level 2 to Level 3
- Transition: All properties at **200ms**

### Input States

**Default**
- Border: 1px solid with border color token
- Background: White or surface color

**Focus**
- Border: 2px solid in accent color
- Box-shadow: `0 0 0 2px var(--color-primary)`
- Transition: **100ms** for immediate feedback

**Error/Invalid**
- Border: 2px solid in danger color
- Box-shadow: `0 1px 1px rgba(0, 0, 0, 0.07), 0 0 0 2px var(--color-danger)`
- Error message appears below input with 8px spacing

---

## Animation & Motion Principles

### Animation Duration Values

The system uses precise timing values calibrated for professional, responsive interactions without feeling sluggish or overwhelming.

- **Extra Fast:** 100ms — Small UI feedback, micro-interactions
- **Fast:** 150ms — Hover states, tooltips
- **Normal (DEFAULT):** 200ms — Standard transitions, most interactions
- **Slow:** 300ms — Modals, larger elements, page content
- **Extra Slow:** 500ms — Complex choreography only (rarely used)

### Easing Curve Specifications

**Default Easing (PRIMARY)**
```css
cubic-bezier(0.4, 0.0, 0.2, 1)
```
Universal easing for most transitions. Provides smooth acceleration and deceleration suitable for professional B2B applications.

**Ease-Out (Entrance)**
```css
cubic-bezier(0.0, 0.0, 0.2, 1)
```
Elements entering the screen or appearing. Starts fast, ends slow.

**Ease-In (Exit)**
```css
cubic-bezier(0.4, 0.0, 1.0, 1.0)
```
Elements leaving the screen or disappearing. Starts slow, ends fast.

### Standard Transition Implementation

```css
transition: transform 200ms cubic-bezier(0.4, 0.0, 0.2, 1),
            opacity 200ms cubic-bezier(0.4, 0.0, 0.2, 1);
```

### When to Animate

**Always Animate:**
- Interactive state changes (hover, focus, active)
- Element entrance and exit
- Property changes users should notice (alerts, updates)
- Navigation transitions
- Loading states and progress indicators

**Never Animate:**
- Initial page load (content appears immediately)
- Text content changes (except opacity)
- Critical data updates requiring immediate attention
- Scrolling (browser handles natively)
- Actions users perform repeatedly in quick succession

### Performance Rules

Only animate **transform** and **opacity** properties as these are GPU-accelerated and don't trigger layout recalculation. Never animate width, height, top, left, margin, or padding. Use `transform: scale()` instead of animating dimensions, and `transform: translate()` instead of positioning properties.

### Page Transition Approach

- Content fade-out: **200ms** ease-in (faster)
- Content fade-in: **300ms** ease-out (slightly slower creates polish)
- No blocking animations — users can interact immediately

### Micro-Interaction Timing

- **Button press:** 100ms or instant with scale(0.98)
- **Tooltip entrance delay:** 200ms (prevents accidental shows)
- **Tooltip entrance:** 150ms
- **Tooltip exit:** 100ms (faster to hide)
- **Dropdown open:** 200ms ease-out
- **Dropdown close:** 150ms ease-in (faster to close)
- **Loading spinner appearance delay:** 300ms (avoids flash for quick loads)

---

## Forms & Input Specifications

### Input Field Specifications

**Height**
- Standard input: **40px** (h-10)
- Small variant: **32px** (h-8)
- Large variant: **48px** (h-12)

**Padding**
- Horizontal: **12px** (left and right)
- Vertical: **8px** (top and bottom)

**Border**
- Default: **1px solid** with border color token
- Border radius: **6px** (medium)
- Focus: **2px** outline or box-shadow ring
- Error: **2px** border in danger color

**Typography**
- Font size: **14px** (minimum 16px on mobile to prevent iOS zoom)
- Font weight: **400**
- Line height: **20px**

### Label Specifications

**Positioning**
- Position above input field (default)
- Spacing from input: **12px** (gap-3)

**Typography**
- Font size: **14px**
- Font weight: **500** (medium for emphasis)
- Line height: **20px**
- Color: Text primary or label color token

**Implementation**
Every input requires an associated `<label>` element for accessibility. Clicking the label focuses the associated control.

### Error State Styling

**Visual Treatment**
- Border: **2px solid** danger color
- Box-shadow: `0 1px 1px rgba(0, 0, 0, 0.07), 0 0 0 2px var(--color-danger)`
- Error message appears immediately below input

**Error Message**
- Font size: **12px**
- Color: Danger color token
- Spacing from input: **8px** margin-top
- Icon: Optional error icon at 16px size

### Helper Text Specifications

- Font size: **12px**
- Color: Muted/secondary text color token
- Spacing from input: **8px** margin-top
- Maximum line length: 100 characters for optimal readability

---

## Additional Foundations

### Icon Sizing Standards

The system uses a 4px-based icon scale optimized for different contexts and ensuring proper alignment with text.

- **12px:** Special cases only (chevrons, small indicators)
- **16px:** DEFAULT — Inline text, buttons, chips, breadcrumbs
- **20px:** Paired with 16px body text, navigation items
- **24px:** Primary actions, larger prominence
- **32px:** High emphasis elements, feature icons

**Touch Targets:** Icons used as interactive elements require minimum 44px touch target area. Add CSS padding to meet requirements without changing visual icon size.

### Image Border Radius Defaults

- **Avatars:** 50% or 9999px (full circle)
- **Thumbnail images:** 4px (small)
- **Card images:** 8px (large) or inherit card radius
- **Hero images:** 12px (xl) or 0px for full-bleed

### Overlay/Backdrop Styling

**Modal and Drawer Backdrops**
- Background color: **rgba(0, 0, 0, 0.5)** — 50% opacity black
- Transition: `opacity 200ms ease`
- Z-index: **40** (overlays sit above content but below modals)
- Position: Fixed, covering entire viewport

### Divider/Separator Specifications

**Visual Treatment**
- Thickness: **1px** solid line
- Color: Border color token (subtle gray)
- Spacing: Minimum 16px from adjacent content

**Implementation**
Use semantic `<hr>` element with `role="separator"`. Horizontal orientation separates vertical content; vertical orientation separates horizontal content.

### Focus Ring Specifications (Accessibility)

**Visual Treatment**
- Width: **2px** solid outline
- Offset: **2px** from element edge (outlineOffset: 2px)
- Color: Primary brand color with minimum 3:1 contrast ratio
- Style: Solid, no dashed borders

**Implementation**
```css
:focus-visible {
  outline: 2px solid var(--color-primary);
  outline-offset: 2px;
}
```

Use `:focus-visible` instead of `:focus` to show focus rings only for keyboard navigation, not mouse clicks.

### Toast/Notification Positioning

**Default Position:** Top-right corner (Windows/modern SaaS standard)

**Spacing**
- Distance from viewport edge: **24px**
- Spacing between stacked toasts: **12px**
- Z-index: **50** (above modals)

**Animation**
- Entrance: Slide from right with 200ms duration
- Auto-dismiss: 4 seconds for informational, persistent for errors
- Exit: Fade out with 150ms duration

**Styling**
- Border radius: **8px** (large)
- Shadow: Level 3 (overlay)
- Padding: **16px** internal
- Max-width: **420px**

---

## Implementation Tokens (CSS Custom Properties)

```css
:root {
  /* Typography */
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-size-base: 16px;
  
  /* Spacing Scale */
  --space-0: 0;
  --space-xs: 0.25rem;   /* 4px */
  --space-sm: 0.5rem;    /* 8px */
  --space-md: 1rem;      /* 16px */
  --space-lg: 1.5rem;    /* 24px */
  --space-xl: 2rem;      /* 32px */
  --space-2xl: 3rem;     /* 48px */
  --space-3xl: 4rem;     /* 64px */
  --space-4xl: 5rem;     /* 80px */
  
  /* Border Radius */
  --radius-sm: 4px;
  --radius-md: 6px;
  --radius-lg: 8px;
  --radius-xl: 12px;
  --radius-2xl: 16px;
  --radius-full: 9999px;
  
  /* Elevation Shadows */
  --shadow-raised: 0 1px 2px rgba(0, 0, 0, 0.04), 0 1px 3px rgba(0, 0, 0, 0.02);
  --shadow-elevated: 0 2px 4px rgba(0, 0, 0, 0.06), 0 1px 2px rgba(0, 0, 0, 0.04);
  --shadow-overlay: 0 4px 8px rgba(0, 0, 0, 0.08), 0 2px 4px rgba(0, 0, 0, 0.06);
  --shadow-modal: 0 8px 16px rgba(0, 0, 0, 0.12), 0 4px 8px rgba(0, 0, 0, 0.08);
  
  /* Animation */
  --duration-fast: 150ms;
  --duration-normal: 200ms;
  --duration-slow: 300ms;
  --easing-default: cubic-bezier(0.4, 0.0, 0.2, 1);
  --easing-out: cubic-bezier(0.0, 0.0, 0.2, 1);
  --easing-in: cubic-bezier(0.4, 0.0, 1.0, 1.0);
  
  /* Container Max-Widths */
  --container-narrow: 640px;
  --container-default: 1140px;
  --container-wide: 1280px;
  --container-full: 1440px;
  
  /* Grid */
  --grid-columns: 12;
  --grid-gutter: 24px;
}
```

---

## Design Principles Summary

**Generous Spacing Philosophy (Stripe-Inspired)**
Default to larger spacing values. Use 24px as the standard component gap, 48-64px for major sections, and never crowd elements. Prioritize breathability and visual clarity over information density.

**Efficient Precision (Linear-Inspired)**
Maintain strict visual alignment using the 8px/4px grid system. Reduce visual noise through subtle shadows and borders. Create clear hierarchy through purposeful spacing contrasts rather than heavy visual elements.

**Data-Driven Aesthetic**
Keep interactions smooth and professional without gimmicks. Use subtle animations (200ms standard) that enhance rather than distract. Maintain high information density where needed while preserving readability through proper typography scale and spacing.

**Performance First**
Only animate GPU-accelerated properties (transform, opacity). Use design tokens exclusively to ensure consistency and enable efficient theme switching. Maintain minimum 16px font size on body text for optimal readability across devices.

**Accessibility Standards**
Meet WCAG 2.1 AA compliance minimum. Provide visible focus indicators on all interactive elements. Ensure proper semantic HTML with ARIA attributes. Support keyboard navigation throughout the interface. Maintain minimum 4.5:1 contrast ratio for text.

---

This specification provides complete, production-ready values for implementing the Oslira design system. All measurements, colors, and interactions are based on proven patterns from Stripe, Linear, Vercel, and Supabase, synthesized into a single coherent system optimized for professional B2B SaaS data platforms.
