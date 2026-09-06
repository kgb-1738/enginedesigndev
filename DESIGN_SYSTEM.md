# Design System — Boot Bodega UI/UX

Boot Bodega's design is **atmospheric archival streetwear** (Nike Football × StockX × Aimé Leon Dore aesthetic). This guide covers color tokens, typography, and component conventions.

## Genre

**Atmospheric Archival Streetwear** — not a generic SaaS dashboard. The product should feel like browsing a curated archive of rare football boots, not administering a system.

### What This Means in Practice

- Dark, warm-toned background (not clinical white)
- Coral accent used sparingly (CTAs, active states, active nav)
- Gold used only for paid/featured items (boosts, premium listings)
- No excess decoration or glass-morphism fluff
- Clear hierarchy: hero → search → results → details

## Color Tokens

| Token | Value | Role | Example |
|---|---|---|---|
| `--bg` | `#0A0A0A` (dark) / `#E4DCC4` (light) | Field / page background | Page body |
| `--text` | `#FFFFFF` (dark) / ink on cream (light) | Primary type | Headlines, body copy |
| `--red` | `#FF000D` | Coral accent | CTAs, active nav, add-to-watch button |
| `--accent` | `#E8C14A` | Gold | Featured listing badge, paid boost indicator |

### Important Constraints

- **No pure `#fff` paper in light mode** — use `#E4DCC4` (warm cream)
- **No yellow hero orbs** — radial underglow is coral only, at **low intensity**
- **No logo-attached glow** — wordmark is clean, no halo or bloom effect
- **Light mode has no hero underglow** — bloom is dark-mode only

## Typography

| Font | Usage | Weights |
|---|---|---|
| **Josefin Sans** | Display (headings, hero tagline) | 300, 400, 600, 700 |
| **Plus Jakarta Sans** | Body (paragraphs, UI copy) | 400, 500, 600 |
| **DM Mono** | Meta (timestamps, prices, labels) | 400, 500 |
| **Bebas Neue** | Numeric only (prices, counts) | 400 |

### Line Heights & Spacing

- **Headings** (H1–H4): 1.2–1.3 line-height, generous margin below
- **Body**: 1.5–1.6 line-height, 16–18px base size
- **Meta/small**: 1.4 line-height, 12–14px size
- **Numeric** (prices): Bebas Neue, no letter spacing, snug vertical rhythm

## Components

### Search Bar (Hero / Results Page)

**Dark mode**:
- Background: `--bg` (#0A0A0A)
- Input text: `--text` (#FFFFFF)
- Border: subtle gray, 1px
- Submit button: `--red` (#FF000D) background, white text
- Placeholder: 50% opacity white

**Light mode**:
- Background: `--bg` (#E4DCC4)
- Input text: ink/dark gray
- Placeholder: 40% opacity dark

### Result Cards

**Hover effect**: Soft editorial blur (CSS filter), not aggressive shadow. Preserve letterbox pad from image when available.

**Card structure**:
```
┌────────────────────┐
│  [Image, padded]   │  ← letterbox margin if image doesn't fill
├────────────────────┤
│ Brand / Model Name │  ← Josefin Sans, bold
│ Retailer · Size    │  ← Plus Jakarta Sans, smaller
│ $150 USD           │  ← Bebas Neue, accent color
│ [Watch] [Zoombar]  │  ← Icon buttons, `--red` on hover
└────────────────────┘
```

### Active States

- **Button/link active**: `--red` background or text
- **Nav active item**: `--red` bottom border or background
- **Watchlist active**: heart icon filled + `--red`

### Featured Listing Badge

- Background: `--accent` (#E8C14A)
- Text: dark (`#0A0A0A`)
- Position: top-right of card
- Font: Josefin Sans, 600 weight, small

### Bottom Dock (Mobile)

**Structure**:
```
┌─────────────────────────────────┐
│ [Search] [Discover] [Sell] [☰]  │  ← floating dock
└─────────────────────────────────┘
  ↑ Glass effect, fixed bottom, pointer-events: none
    (child buttons have pointer-events: auto)
```

**Design**:
- Semi-transparent background with backdrop blur
- Stays visible above mobile keyboard (uses `visualViewport` API)
- Icon + text labels, centered
- Outline focus style for accessibility (`:focus-visible`)

## Chrome & Layout

### Wordmark & Topbar

**Wordmark**: **BOOTEGA | Boot Search Engine** (left-aligned, top-left)
- Font: Josefin Sans, 700, medium size
- No crest logo, no glow

**Topbar** (right-aligned, top-right):
- Account icon (avatar or "Log in")
- Cart/saved icon (unified badge for watch + basket count)
- Theme toggle (light/dark)
- **No logo in topbar** — all chrome is seamless

### Primary Navigation

**Floating dock with 4 clusters**:
1. Search (magnifying glass)
2. Discover / Sell / Bootega (cluster with labels visible on hover)
3. Info (submenu: FAQ, Reviews, Privacy, Terms)

**Behavior**:
- On mobile: always visible, floats above content
- On desktop: slides in/out on scroll (optional, for clean hero)
- Active item: `--red` underline or background

### Search Results Page Layout

**Top**: Translucent search bar (results only, not hero)
**Left**: Filter panel (size, country, currency, display, sort)
**Center**: Grid of result cards (responsive: 1–4 columns)
**Bottom**: Glass fade (semi-transparent overlay, merges with floating dock)
**Right** (wide screens only): Featured rail (paid/boosted listings)

## Tone

- **Simplistic** — clear hierarchy, no visual clutter
- **Modern** — current design patterns, responsive, touch-friendly
- **Sleek** — smooth transitions, restrained animations, clear affordances
- **Clear controls over decorative glass** — blur/opacity only where it aids clarity, not for decorative effect

## Accessibility

- **Focus states** (`:focus-visible`): clear outline, preferably in `--red`
- **Contrast**: 4.5:1 minimum for body text (WCAG AA)
- **Color alone**: don't use color as the only differentiator (use icons + color)
- **Mobile**: buttons min 44×44px touch target

## Animations & Transitions

- **Transitions**: 200–300ms, use `cubic-bezier(0.4, 0, 0.2, 1)` (material curve)
- **Avoid**: `transition: all` (use specific properties like `transition: background-color, color`)
- **Hover effects**: subtle color shift or slight scale (1.02–1.05)
- **Page transitions**: fade in (200ms opacity), no slide/scale distraction

## Component Variants

### Button States

| State | Dark Mode | Light Mode |
|---|---|---|
| Default | `#FF000D` bg, white text | `#FF000D` bg, white text |
| Hover | darker red (`#CC000B`), cursor pointer | darker red (`#CC000B`) |
| Active | `#990007`, white text | `#990007`, white text |
| Disabled | gray background, 50% opacity | gray background, 50% opacity |

### Links

- **Idle**: `--text` with underline (or no underline, depending on context)
- **Hover**: `--red` color, slight underline/weight shift
- **Active/visited**: `--red` color, slightly darker

## Dark & Light Mode

The entire design system must work in both modes. Use CSS custom properties:

```css
:root {
  --bg: #0A0A0A;
  --text: #FFFFFF;
  --red: #FF000D;
  --accent: #E8C14A;
}

@media (prefers-color-scheme: light) {
  :root {
    --bg: #E4DCC4;
    --text: #1a1a1a;  /* ink on cream */
  }
}
```

**Test both modes** with actual system preference, not just a toggle.

## Common Pitfalls

1. **Hero underglow in light mode** — don't. Coral glow is dark-mode only.
2. **Pure white anywhere** — use cream (`#E4DCC4`) in light mode for warmth
3. **Overusing gold** — only for featured/paid items. Everything else uses coral or neutral.
4. **Glass-morphism creep** — blur + transparency aids clarity (dock over content), but doesn't decorate. Use sparingly.
5. **Inconsistent transitions** — stick to 200–300ms, consistent easing curve
6. **Skipping focus states** — every interactive element needs `:focus-visible`

## References

- **Color values**: Verified against brand guidelines (Nike Football aesthetic)
- **Typography stack**: Covers global font availability + fallbacks to sans-serif
- **Component specs**: Derived from Figma designs + QA testing on mobile (iOS Safari, Android Chrome)
