# Design System — Solen Atelier

Warm, artisanal, boutique-luxury feel — an independent high-end printmaking atelier tailored for Iraq's creator economy, not a sterile SaaS marketplace.

---

## 1. Brand Motif & Aesthetic Identity

- **Brand Mark — Sol & Selene (`solen-logo.svg`)**:
  - **Sol (The Sun)**: Radiant ochre gold core (`#C8A46A`) representing heat-press curing, solar ink vitality, and daytime creator energy.
  - **Selene (The Moon)**: Archival crescent silhouette embracing the sun, representing reflective nocturnal inspiration, quiet craft, and Mesopotamian celestial heritage.
  - **Printmaker Guild Seal**: An outer circular relief rim with four cardinal CMYK/DTF registration cross-hairs and triple celestial linocut stars, symbolizing print precision.
- **Style**: **Soft Tactile Neumorphism**: paired opposing dual shadows (ambient dark falloff + warm luminous highlight) over textured organic tones. Never flat, never razor-bordered.
- **Light Source**: Fixed top-left (~135°). Consistent directional lighting across both LTR and RTL layouts prevents visual dissonance.
- **Corner Radius**: 10–14px (`rounded-xl` to `rounded-2xl`) for tactile, crafted ergonomics.

---

## 2. Typography & Numerals Standard

### Font Families
- **Arabic (Primary)**: **IBM Plex Sans Arabic** across all weights (300 Light, 400 Regular, 500 Medium, 600 SemiBold, 700 Bold). Used for headlines, UI controls, and body copy in Arabic.
- **Latin Display/Headlines**: **Playfair Display** (editorial, classical printmaking serif tone).
- **Latin Body/UI**: **IBM Plex Sans** or **Plus Jakarta Sans** (clean, crisp legibility).

### Strict Numerals Invariant
> [!IMPORTANT]
> **Always use Western Arabic numerals (`1, 2, 3, 4, 5, 6, 7, 8, 9, 0`) across all locales.**
> Never convert numbers into Eastern Arabic digits (`١, ٢, ٣`). This applies to prices (e.g. `25,000 د.ع`), measurements (e.g. `30 × 40 cm`), and phone numbers.

---

## 3. Color Tokens

### Light Mode
| Token | Hex | Role |
|---|---|---|
| `--bg-scaffold` | `#F4EFE8` | Warm parchment scaffold background |
| `--bg-surface` | `#F4EFE8` | Elevated tactile cards and surfaces |
| `--bg-surface-recessed` | `#F0EAE1` | Inset inputs, slider tracks, pressed state |
| `--bg-canvas` | `#FFFFFF` | Fabric.js editor artboard & flat template |
| `--primary-accent` | `#C8A46A` | Atelier gold / warm ochre accent |
| `--primary-accent-hover` | `#B89358` | Deepened gold hover state |
| `--text-on-primary` | `#2C2A2A` | High-contrast dark charcoal on gold |
| `--text-primary` | `#2C2A2A` | Deep charcoal primary typography |
| `--text-secondary` | `#605D5D` | Warm muted gray secondary typography |
| `--text-tertiary` | `#8C867D` | Subtle tertiary & hint text |
| `--border-subtle` | `#D9D5CF` | Subtle structural dividers |
| `--border-standard` | `#B8ACA0` | Defined borders |
| `--chip-bg` | `#DCD4C7` | Tag and attribute chip background |

### Dark Mode
| Token | Hex | Role |
|---|---|---|
| `--bg-scaffold` | `#2C2A2A` | Deep charcoal studio scaffold |
| `--bg-surface` | `#373535` | Elevated dark tactile surface |
| `--bg-surface-recessed` | `#232222` | Inset dark inputs and recessed states |
| `--bg-canvas` | `#1F1E1E` | Dark studio canvas backing |
| `--primary-accent` | `#C8A46A` | Atelier gold accent |
| `--primary-accent-hover` | `#D6B37A` | Luminous gold hover state |
| `--text-on-primary` | `#2C2A2A` | Dark charcoal on gold |
| `--text-primary` | `#F4EFE8` | Warm parchment primary text |
| `--text-secondary` | `#C7BEB5` | Muted parchment secondary text |
| `--text-tertiary` | `#8F877F` | Subtle dark-mode hint text |
| `--border-subtle` | `#423E3E` | Deep subtle divider |
| `--border-standard` | `#4A4747` | Defined dark border |
| `--chip-bg` | `#3E3C3C` | Dark chip surface |

---

## 4. Shadow Tokens (Soft Neumorphic Elevation)

### Light Mode
```css
--shadow-raised: 6px 6px 16px rgba(184,172,160,0.45), -6px -6px 16px rgba(255,255,255,0.85);
--shadow-raised-sm: 4px 4px 10px rgba(184,172,160,0.40), -4px -4px 10px rgba(255,255,255,0.80);
--shadow-floating: 10px 10px 25px rgba(184,172,160,0.50), -10px -10px 25px rgba(255,255,255,0.90);
--shadow-recessed: inset 3px 3px 7px rgba(184,172,160,0.40), inset -3px -3px 7px rgba(255,255,255,0.70);
--shadow-gold: 4px 4px 14px rgba(200,164,106,0.45), -2px -2px 10px rgba(255,255,255,0.60);
```

### Dark Mode
```css
--shadow-raised: 6px 6px 18px rgba(18,17,17,0.75), -5px -5px 15px rgba(255,255,255,0.06);
--shadow-raised-sm: 4px 4px 10px rgba(18,17,17,0.65), -3px -3px 9px rgba(255,255,255,0.05);
--shadow-floating: 10px 10px 30px rgba(15,14,14,0.85), -6px -6px 20px rgba(255,255,255,0.07);
--shadow-recessed: inset 3px 3px 6px rgba(14,13,13,0.80), inset -2px -2px 6px rgba(255,255,255,0.04);
--shadow-gold: 0 6px 20px rgba(200,164,106,0.35), inset 0 1px 1px rgba(255,255,255,0.25);
```

---

## 5. Component Patterns

- **Primary Button (`.btn-primary`)**: Solid `#C8A46A` gold background, `#2C2A2A` text, font weight 600, 12px radius, `shadow-gold`. On click/active: `scale(0.98)` and shadow compresses.
- **Secondary Tactile Button (`.btn-secondary`)**: Background `bg-surface`, 1px `border-subtle`, `shadow-raised-sm`, `text-primary`.
- **Recessed Input (`.input-recessed`)**: Background `bg-surface-recessed`, `shadow-recessed`, 12px radius, visible focus outline that does not rely on shadow contrast alone.
- **Color Swatches**: 30px circular pill, 2px white rim, raised ring. Active state: gold outline + checkmark.
- **Size & Placement Chips**: Raised tactile button by default; active state inverts to recessed with gold text indicator.
- **Floating Nav Dock**: Pill container, `rounded-2xl`, `shadow-floating`, backdrop blur.

---

## 6. Bidirectionality & RTL Rules

- Root level `dir="rtl"` applied whenever `locale === 'ar'` or `locale === 'ku'`.
- Use logical Tailwind CSS utilities (e.g. `ms-4`, `me-4`, `ps-4`, `pe-4`, `start-0`, `end-0`) rather than physical left/right.
- Do not reverse shadow light angles in RTL: a consistent global light source at 135° looks natural and grounded across all languages.

---

## 7. Mobile-First Responsiveness Standards

- **Touch Targets**: Minimum 44px height/width for all touch interactions.
- **Adaptive Layouts**: Action buttons must adapt to full width on compact mobile viewports (`w-full sm:w-auto`).
- **Safe Area Insets**: Floating docks and action bars must respect `env(safe-area-inset-bottom)`.
- **Viewports**: All designs tested from 320px–430px mobile up through 1280px+ desktop without horizontal scroll overflow.
