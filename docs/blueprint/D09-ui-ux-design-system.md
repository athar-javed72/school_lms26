# Deliverable 9: UI/UX Design System

**Version:** 1.0 (draft for approval) | **Date:** 24 Nov 2026
**Builds on:**
- D1: token names (1.16), white-labeling (1.2), principles PR-05 to PR-07, DEC-05, DEC-06, DEC-11
- D2: FR-GEN-007, FR-GEN-010, FR-GEN-011, FR-GEN-016 to FR-GEN-018
- D3: frontend architecture (3.17), Alpine CSP build (DEC-30)
- D8: component folders and bundles (8.6, 8.9)

Every HEX value and contrast ratio in this document was computed, not estimated. Contrast follows WCAG 2.2 relative luminance. Chart palettes were run through a colour-vision-deficiency validator against this system's own surfaces.

---

## 9.1 New Decisions

| ID | Decision |
|---|---|
| DEC-138 | **Three token layers.** (1) Primitive scales: eight hues with eleven steps each. (2) Role tokens: the D1 names plus role variants such as `success-tint`. (3) Component tokens for the few places that need them, such as navigation. Blade views and components use **role tokens only**. Numbered steps appear only in `tokens.css`, the chart theme and the branding generator. |
| DEC-139 | **Tailwind's default palette is switched off** (`--color-*: initial` in `@theme`). Only this system's tokens exist, so `bg-teal-600` or a raw HEX in a view can't compile into a colour. |
| DEC-140 | **Colour usage rule.** Fills (buttons, solid badges) use step 600 with white text. Coloured text on a page uses step 700 in light mode. Tinted backgrounds use step 50 with step 800 text. The role tokens encode this, so developers don't pick steps. |
| DEC-141 | **Tenant brand colour drives the `accent` scale only.** The server generates the scale in OKLCH from the organization's colour and checks four contrast gates (9.20). Primary actions and status colours never change per tenant, so "red means error" holds in every school. |
| DEC-142 | **Screen text uses the system font stack**, which costs no download. Urdu uses self-hosted Noto Nastaliq Urdu, loaded through `unicode-range` so it downloads only when Urdu characters appear on the page. Users can switch Urdu to Noto Naskh Arabic, which is lighter and faster on budget phones. PDFs embed Noto Sans and both Urdu fonts inside the Gotenberg image. |
| DEC-143 | **Numbers, codes, phone numbers, CNIC, money and dates are always left-to-right and isolated** (`<bdi dir="ltr">`), with Western digits (FR-GEN-007) and tabular figures. This keeps them readable inside Urdu sentences. |
| DEC-144 | **Dark theme.** Role tokens for dark mode are defined and contrast-checked now. The theme ships in R2 for the staff, guardian and student portals, selected per user (`system`, `light`, `dark`). Print and PDF output are always light. |
| DEC-145 | **Icons** are a fixed subset of Lucide (ISC licence) stored as SVG files and rendered inline by `x-ui.icon`. No icon font, no icon CDN. Directional icons mirror in RTL. |
| DEC-146 | **Density.** The staff portal offers `comfortable` (default) and `compact` table density per user. Guardian and student portals have one comfortable density. |
| DEC-147 | **Hide or disable.** An action the user has no permission for is **hidden**. An action the user may perform but that the record's state blocks is **disabled with a visible reason** (for example "Locked: November attendance is closed"). |
| DEC-148 | **Status is never shown by colour alone.** Every status pill carries text, and attendance cells always show the code letter. This also makes printouts work in black and white. |
| DEC-149 | **Chart colours** use a validated eight-slot categorical palette in fixed order, with separate light and dark steps. Scatter-type charts cap at three series. Status colours are never used as series colours. |
| DEC-150 | **The platform portal looks different on purpose:** a near-black top bar with a "Platform" badge. Support impersonation shows a persistent banner on every page until it ends. |

### Corrections to earlier deliverables

| Where | Change |
|---|---|
| D8 8.9 `resources/fonts` | Holds only the Urdu web fonts (Noto Nastaliq Urdu, Noto Naskh Arabic). Latin screen text uses system fonts (DEC-142). PDF fonts live in the Gotenberg image. |
| D8 8.9 `components/ui` list | Adds `banner`, `chip`, `description-list`, `progress`, `stepper`, `tooltip`, `filter-bar` and `bulk-bar` (9.15). |

---

## 9.2 Design Principles

| # | Principle | What it means on screen |
|---|---|---|
| UX-01 | Today first | Every dashboard opens with what needs doing today: unmarked attendance, pending approvals, today's collection, overdue items. |
| UX-02 | One primary action per screen | One `primary` button per page header. Everything else is secondary, ghost or in a menu. |
| UX-03 | Show the scope | The current branch, session and (for guardians) the selected child are always visible in the top bar. |
| UX-04 | Explain blocked actions | Disabled controls say why (DEC-147). Locked periods, closed sessions and pending approvals are shown where the user acts. |
| UX-05 | Protect history | Destructive and protected actions confirm, ask for a reason when the audit needs one, and say what will happen. |
| UX-06 | Fast on a cheap phone | No web font for Latin text, no images except photos and logos, small JS bundles, skeletons instead of spinners. |
| UX-07 | Urdu is equal | Every screen is designed and tested in RTL. Layout uses logical properties only. |
| UX-08 | Keyboard and screen reader complete | Every action is reachable by keyboard; every control has a label; focus is always visible. |
| UX-09 | Same thing, same look | A voucher status looks identical in the list, the detail page, the guardian portal and the PDF. |
| UX-10 | Calm by default | Neutral surfaces, colour only for meaning and the primary action. No decoration that competes with data. |

---

## 9.3 Colour: Primitive Scales

OKLCH-generated scales, gamut-clamped to sRGB. The number in brackets is the contrast ratio against white `#FFFFFF`.

| Step | primary (teal) | secondary (navy) | accent (violet) | neutral (slate) |
|---|---|---|---|---|
| 50 | `#E7FDFA` (1.06) | `#F4F8FE` (1.07) | `#F8F7FE` (1.06) | `#F6F8FA` (1.06) |
| 100 | `#D1F9F3` (1.13) | `#E7F0FE` (1.15) | `#F0ECFE` (1.16) | `#EDEFF3` (1.15) |
| 200 | `#A8EFE6` (1.30) | `#CCE0FE` (1.34) | `#E1D9FE` (1.35) | `#DADFE5` (1.34) |
| 300 | `#72DED2` (1.60) | `#AECAF4` (1.67) | `#CCBCFE` (1.72) | `#C2C9D1` (1.67) |
| 400 | `#30C5B8` (2.14) | `#8DAEDF` (2.27) | `#B498FE` (2.37) | `#A5ADB7` (2.27) |
| 500 | `#07A79B` (3.00) | `#7092C5` (3.18) | `#9C70FB` (3.43) | `#89919C` (3.18) |
| 600 | `#088178` (4.75) | `#5071A1` (4.98) | `#7C4CD5` (5.45) | `#69717B` (4.94) |
| 700 | `#067169` (5.88) | `#44638F` (6.13) | `#6D3FC0` (6.73) | `#5B626C` (6.16) |
| 800 | `#005B54` (8.00) | `#354F76` (8.30) | `#59309F` (8.94) | `#494F57` (8.27) |
| 900 | `#044742` (10.58) | `#283E5D` (10.84) | `#45247F` (11.59) | `#393E44` (10.79) |
| 950 | `#002E2A` (14.74) | `#18273D` (15.04) | `#2C1455` (15.69) | `#23272B` (15.04) |
| 975 | | | | `#16191C` (dark canvas only) |

| Step | success (green) | warning (amber) | danger (red) | info (blue) |
|---|---|---|---|---|
| 50 | `#EAFEED` (1.06) | `#FEF6ED` (1.07) | `#FFF5F4` (1.07) | `#F3F8FF` (1.07) |
| 100 | `#D6FADC` (1.13) | `#FFECD7` (1.15) | `#FEEAE8` (1.16) | `#E5F1FE` (1.14) |
| 200 | `#B2F1BE` (1.29) | `#FFD7AA` (1.35) | `#FFD3CE` (1.36) | `#C9E1FE` (1.34) |
| 300 | `#85E099` (1.60) | `#FFB860` (1.71) | `#FEB1AA` (1.74) | `#A0CCFE` (1.67) |
| 400 | `#55C875` (2.12) | `#EB9812` (2.32) | `#FF817A` (2.42) | `#6AB1FE` (2.25) |
| 500 | `#21AC54` (2.96) | `#C67F09` (3.26) | `#F24F4E` (3.50) | `#3093F4` (3.18) |
| 600 | `#00873C` (4.63) | `#9B6200` (5.07) | `#CB242D` (5.48) | `#0371CA` (4.98) |
| 700 | `#027634` (5.76) | `#875505` (6.30) | `#B51522` (6.78) | `#0162B2` (6.19) |
| 800 | `#015F29` (7.86) | `#6D4403` (8.47) | `#960416` (9.03) | `#014F91` (8.31) |
| 900 | `#014B1F` (10.36) | `#563502` (11.02) | `#77020F` (11.67) | `#013E72` (10.86) |
| 950 | `#013011` (14.66) | `#382101` (15.15) | `#4F0107` (15.67) | `#01274B` (15.07) |

**Why these hues:** teal for primary is calm, reads as "education", and stays clearly apart from status green (about 36° of hue apart, and success always carries an icon). Navy anchors the staff navigation. Violet is the default accent because it collides with no status colour; each organization replaces it with its own brand colour (9.20).

---

## 9.4 Colour: Role Tokens

These are the only colour names used in views. `R` stands for each role: `primary`, `secondary`, `accent`, `success`, `warning`, `danger`, `info`.

### Surfaces and text

| Token | Light | Dark (R2) | Use | Verified contrast |
|---|---|---|---|---|
| `canvas` | `#F6F8FA` (neutral-50) | `#16191C` (neutral-975) | Page background | |
| `surface` | `#FFFFFF` | `#23272B` (neutral-950) | Cards, tables, forms, modals | |
| `surface-muted` | `#EDEFF3` (neutral-100) | `#393E44` (neutral-900) | Table headers, wells, read-only fields | |
| `surface-raised` | `#FFFFFF` + shadow | `#393E44` (neutral-900) | Dropdowns, popovers, toasts | |
| `border` | `#DADFE5` (neutral-200) | `#494F57` (neutral-800) | Decorative dividers and card edges | Decorative, no minimum |
| `border-strong` | `#89919C` (neutral-500) | `#89919C` (neutral-500) | Input, checkbox and select borders | 3.18 on white; 4.72 on dark surface (WCAG 1.4.11 needs 3:1) |
| `ink` | `#23272B` (neutral-950) | `#F6F8FA` (neutral-50) | Body text, headings | 15.04 on surface, 14.12 on canvas; dark 14.12 |
| `ink-muted` | `#5B626C` (neutral-700) | `#A5ADB7` (neutral-400) | Secondary text, help text, table headers | 6.16 on surface, 5.79 on canvas, 5.35 on surface-muted; dark 6.63 on surface, 4.76 on surface-muted |
| `ink-subtle` | `#69717B` (neutral-600) | `#89919C` (neutral-500) | Placeholders, timestamps | 4.94 on white; dark 4.72 |
| `ink-disabled` | `#A5ADB7` (neutral-400) | `#5B626C` (neutral-700) | Disabled labels | Exempt under WCAG; always paired with a reason (DEC-147) |
| `ink-inverse` | `#FFFFFF` | `#23272B` | Text on fills | |
| `focus` | `#088178` (primary-600) | `#72DED2` (primary-300) | 2px focus ring with 2px offset | 4.75 on white, 4.46 on canvas (3:1 needed) |
| `overlay` | `rgb(35 39 43 / 0.55)` | `rgb(0 0 0 / 0.65)` | Modal and drawer backdrop | |
| `selection` | `#D1F9F3` (primary-100) | `#044742` (primary-900) | Selected table rows, text selection | Ink on it: 13.29 |

### Role variants

| Token pattern | Light step | Dark step | Use |
|---|---|---|---|
| `R` | 600 | 600 | Solid fill with `ink-inverse` text (buttons, solid badges, icons) |
| `R-strong` | 700 | 700 | Hover and pressed fill |
| `R-text` | 700 | 300 | Coloured text and links on `surface` or `canvas` |
| `R-tint` | 50 | 950 | Tinted background for alerts, pills, selected states |
| `R-on-tint` | 800 | 200 | Text and icons on `R-tint` |
| `R-border` | 200 | 800 | Border of tinted containers |

**Verified pairs (every role):**

| Role | White on `R` (600) | `R-text` on white (700) | `R-text` on canvas (700) | `R-on-tint` on `R-tint` (800 on 50) | Dark `R-text` on dark surface (300) | Dark `R-on-tint` on `R-tint` (200 on 950) |
|---|---|---|---|---|---|---|
| primary | 4.75 | 5.88 | 5.55 | 7.56 | 9.38 | 11.34 |
| secondary | 4.98 | 6.13 | 5.75 | 7.78 | 6.45 | n/a |
| accent | 5.45 | 6.73 | 6.32 | 8.40 | 6.26 | n/a |
| success | 4.63 | 5.76 | 5.46 | 7.45 | 9.39 | 11.32 |
| warning | 5.07 | 6.30 | 5.89 | 7.92 | 8.79 | 11.22 |
| danger | 5.48 | 6.78 | 6.34 | 8.44 | 8.65 | 11.54 |
| info | 4.98 | 6.19 | 5.80 | 7.78 | 9.00 | 11.26 |

All text pairs meet 4.5:1 (normal text, WCAG 1.4.3). All fills and focus rings meet 3:1 against their background (WCAG 1.4.11).

**Exception, `secondary`:** the secondary role is navy chrome, not an action colour. `secondary` maps to `#283E5D` (secondary-900) in light mode and `#18273D` (secondary-950) in dark mode, and is used for the staff navigation and the print header band. Secondary *buttons* are neutral outline buttons (9.15).

### Component tokens: navigation

| Token | Light and dark | Value | Verified |
|---|---|---|---|
| `nav-bg` | Light | `#283E5D` (secondary-900) | |
| | Dark | `#18273D` (secondary-950) | |
| `nav-ink` | Both | `#E7F0FE` (secondary-100) | 9.44 on light nav, 13.10 on dark nav |
| `nav-ink-muted` | Both | `#AECAF4` (secondary-300) | 6.48 on light nav, 8.99 on dark nav |
| `nav-active-bg` | Both | `#354F76` (secondary-800) | White text on it 8.30 |
| `nav-indicator` | Both | `accent-400` (`#B498FE` by default) | 4.57 on light nav, 6.35 on dark nav; tenant colours must pass 3:1 (9.20) |
| `platform-bar` | Both | `#16191C` (neutral-975) | Platform portal only (DEC-150) |

---

## 9.5 Status Colour Mapping

Every status uses a pill: tint background, on-tint text, an icon or dot, and the label (DEC-148). The same mapping is used on screen, in exports and in PDFs (PDFs use text only).

### Generic workflow statuses

| Status values | Role | Icon |
|---|---|---|
| `draft` | neutral (`surface-muted` / `ink-muted`) | `file-pen` |
| `pending`, `submitted`, `issued`, `scheduled` | info | `clock` |
| `under_review`, `returned`, `partially_paid`, `compartment`, `on_hold` | warning | `alert-triangle` |
| `approved`, `published`, `paid`, `active`, `pass`, `completed`, `resulted` | success | `check-circle` |
| `rejected`, `overdue`, `fail`, `failed`, `bounced` | danger | `x-circle` |
| `cancelled`, `superseded`, `archived`, `withdrawn`, `expired`, `inactive` | neutral | `circle-slash` |
| `locked`, `withheld`, `closed` | secondary (navy tint `#F4F8FE` / `#354F76`) | `lock` |

### Attendance codes (D6, DEC-24, DEC-73)

| Code | Meaning | Role | Cell rendering |
|---|---|---|---|
| P | Present | success | Letter `P` on `success-tint` |
| A | Absent | danger | Letter `A` on `danger-tint`, bold |
| L | Late | warning | Letter `L` on `warning-tint` |
| HD | Half day | warning | `HD` on `warning-tint` with a diagonal half fill |
| LV | Leave | info | `LV` on `info-tint` |
| H | Holiday | neutral | `H` on `surface-muted` |
| W | Weekend | neutral | `W` in `ink-subtle`, no fill |
| NM | Not marked | neutral | `NM` in `ink-muted` with a dashed `border-strong` outline |
| SU | Suspended | secondary | `SU` on navy tint |

### Fee voucher statuses (D7 7.26)

| Status | Role | Label (en) |
|---|---|---|
| `issued` | info | Issued |
| `partially_paid` | warning | Partly paid |
| `paid` | success | Paid |
| `overdue` | danger | Overdue |
| `cancelled` | neutral | Cancelled |
| `superseded` | neutral | Replaced |

Labels come from each enum's `label()` (D3 3.6), so wording can be changed by translation overrides without code changes.

---

## 9.6 Typography

### Font stacks

| Token | Stack | Use |
|---|---|---|
| `--font-sans` | `"Darsgah Urdu", system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans", "Helvetica Neue", Arial, sans-serif` | All UI text. Latin falls through to the system font; Urdu characters use the Urdu face (unicode-range). |
| `--font-numeric` | `system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans", Arial, sans-serif` | Isolated numbers inside Urdu text (DEC-143) |
| `--font-mono` | `ui-monospace, "SFMono-Regular", Menlo, Consolas, monospace` | API keys, device serials, technical IDs only |

`"Darsgah Urdu"` is an alias defined twice, and the layout picks one by the user's preference (`data-urdu-font` on `<html>`):

```css
@font-face {
  font-family: "Darsgah Urdu Nastaliq";
  src: url("/build/fonts/NotoNastaliqUrdu-Regular.woff2") format("woff2");
  font-weight: 400 700;
  font-display: swap;
  unicode-range: U+0600-06FF, U+0750-077F, U+08A0-08FF, U+FB50-FDFF, U+FE70-FEFF, U+200C-200F;
}
@font-face {
  font-family: "Darsgah Urdu Naskh";
  src: url("/build/fonts/NotoNaskhArabic-Regular.woff2") format("woff2");
  font-weight: 400 700;
  font-display: swap;
  unicode-range: U+0600-06FF, U+0750-077F, U+08A0-08FF, U+FB50-FDFF, U+FE70-FEFF, U+200C-200F;
}
```

The `--font-sans` value starts with `var(--font-urdu-face)`, which is `"Darsgah Urdu Nastaliq"` by default and `"Darsgah Urdu Naskh"` when `data-urdu-font="naskh"`.

### Type scale (Latin)

| Token | Size / line height | Weight | Use |
|---|---|---|---|
| `text-xs` | 12 / 16 px | 500 | Pill labels, table meta, captions |
| `text-sm` | 14 / 20 px | 400 | Staff tables, help text, secondary text |
| `text-base` | 16 / 24 px | 400 | Body, all inputs on mobile, portal text |
| `text-lg` | 18 / 28 px | 600 | Card titles, section headings |
| `text-xl` | 20 / 28 px | 600 | Page titles on mobile |
| `text-2xl` | 24 / 32 px | 600 | Page titles on desktop |
| `text-3xl` | 30 / 36 px | 600 | Stat tile values |
| `text-4xl` | 36 / 44 px | 700 | Hero number (one per dashboard) |

Weights: 400 regular, 500 medium (labels, buttons), 600 semibold (headings, table headers), 700 bold (hero numbers, absent `A`). Italics are not used.

### Urdu typography

Nastaliq is tall and slanted; it needs more size and much more line height than Latin text.

| Rule | Value |
|---|---|
| Size | Urdu text renders one step larger: `:lang(ur)` sets `font-size: 1.125em` |
| Line height | Body 2.0, headings 1.8, table cells 1.9 |
| Minimum row height | Table rows 48px in Urdu (36px compact does not apply to Urdu) |
| Letter spacing | Never applied to Urdu |
| Case transforms | `uppercase`, `capitalize` never used anywhere (they break Urdu and look shouty in English) |
| Truncation | Prefer wrapping; when truncation is unavoidable, use line clamp, never mid-word ellipsis on a single line |
| Mixed content | Urdu names inside English pages carry `lang="ur" dir="rtl"` on their element so the right font, size and direction apply |

### Numbers

- `tabular-nums` on every number in a table column, stat tile, money value, mark and date.
- Money, marks and counts are end-aligned in tables (right in LTR, left in RTL).
- Numbers inside Urdu text are wrapped in `<bdi dir="ltr" class="font-numeric">` by the `x-ui.money`, `x-ui.date` and `@number` helpers, never by hand.

---

## 9.7 Spacing, Sizing and Layout

### Spacing

Tailwind's 4px base (`--spacing: 0.25rem`). Allowed steps in layouts: 1, 2, 3, 4, 6, 8, 10, 12, 16 (4px to 64px).

| Use | Value |
|---|---|
| Inside a pill | 2px block, 8px inline |
| Inside a button | 8px block, 16px inline (md) |
| Inside an input | 8px block, 12px inline |
| Card padding | 16px mobile, 24px desktop |
| Gap between form fields | 16px; between form sections 32px |
| Page padding | 16px mobile, 24px tablet, 32px desktop |
| Gap between cards in a grid | 16px mobile, 24px desktop |

### Fixed sizes

| Element | Size |
|---|---|
| Top bar | 56px high |
| Staff sidebar | 264px expanded, 72px collapsed (icons with tooltips) |
| Portal bottom navigation | 64px high, 5 items maximum |
| Content width, staff | Fluid up to 1440px (`--container-app: 90rem`) |
| Content width, portals | 640px centred (`--container-portal: 40rem`) |
| Form width | 768px maximum (`--container-form: 48rem`) |
| Modal widths | sm 400px, md 560px, lg 720px, full screen below 640px |
| Drawer width | 400px (filters), 560px (record preview); full width below 640px |

### Touch and click targets

| Context | Minimum |
|---|---|
| Portals and any screen below 768px | 44 x 44 px |
| Staff desktop controls | 32px high, with at least 24 x 24 px hit area and 8px spacing (WCAG 2.5.8) |
| Attendance marking cells | 44 x 44 px on every screen size |

### Grid

| Breakpoint | Columns | Gutter |
|---|---|---|
| Below 768px | 4 | 16px |
| 768 to 1023px | 8 | 24px |
| 1024px and up | 12 | 24px |

---

## 9.8 Radius, Elevation and Layers

### Radius

| Token | Value | Use |
|---|---|---|
| `rounded-sm` | 4px | Checkboxes, small tags |
| `rounded-md` | 6px | Buttons, inputs, selects |
| `rounded-lg` | 8px | Cards, tables, alerts |
| `rounded-xl` | 12px | Modals, drawers, bottom sheets |
| `rounded-full` | 9999px | Pills, avatars, toggles |

### Shadows (light theme)

| Token | Value | Use |
|---|---|---|
| `shadow-xs` | `0 1px 2px rgb(35 39 43 / 0.06)` | Inputs, buttons |
| `shadow-sm` | `0 1px 3px rgb(35 39 43 / 0.10), 0 1px 2px rgb(35 39 43 / 0.06)` | Cards |
| `shadow-md` | `0 4px 8px -2px rgb(35 39 43 / 0.10), 0 2px 4px -2px rgb(35 39 43 / 0.06)` | Dropdowns, sticky bars |
| `shadow-lg` | `0 12px 24px -6px rgb(35 39 43 / 0.18)` | Modals, drawers, toasts |

In dark theme, shadows are replaced by a 1px `border` and the lighter `surface-raised`.

### Layers (z-index)

| Token | Value | Element |
|---|---|---|
| `z-sticky` | 10 | Sticky table headers, sticky form action bar |
| `z-topbar` | 20 | Top bar, impersonation banner |
| `z-sidebar` | 30 | Sidebar and mobile navigation drawer |
| `z-dropdown` | 40 | Menus, selects, date picker |
| `z-drawer` | 50 | Filter and preview drawers |
| `z-modal` | 60 | Modals and confirm dialogs |
| `z-toast` | 70 | Toasts |
| `z-tooltip` | 80 | Tooltips |

---

## 9.9 Breakpoints and Responsive Rules

Tailwind defaults: `sm` 640px, `md` 768px, `lg` 1024px, `xl` 1280px, `2xl` 1536px. Design starts at **360px** (FR-GEN-017).

| Element | Below `md` (768) | `md` to `lg` | `lg` and up |
|---|---|---|---|
| Staff navigation | Drawer from the start edge | Drawer | Fixed sidebar, collapsible |
| Portal navigation | Bottom bar | Bottom bar | Top tabs in a 640px column |
| Data tables | Card list (9.16) | Table with priority columns only | Full table |
| Filters | Search plus "Filters" button opening a drawer | Search plus 2 inline filters | Search plus up to 3 inline filters |
| Page header actions | Primary action full width below the title; others in a menu | Inline | Inline |
| Modals | Full-screen sheet | Centred | Centred |
| Forms | One column | One column | Two columns for short paired fields only |

---

## 9.10 Motion

| Token | Value | Use |
|---|---|---|
| `--ease-standard` | `cubic-bezier(0.2, 0, 0, 1)` | Everything |
| `--duration-fast` | 120ms | Hover, pressed, colour changes |
| `--duration-base` | 180ms | Dropdowns, tooltips, toasts in |
| `--duration-slow` | 240ms | Drawers, modals, bottom sheets |

Rules: no motion longer than 240ms; nothing loops except loading indicators; `prefers-reduced-motion: reduce` removes all movement and keeps only opacity fades of 120ms.

---

## 9.11 Iconography

| Rule | Value |
|---|---|
| Set | Lucide, fixed subset copied to `resources/svg/icons/{name}.svg` |
| Rendering | `<x-ui.icon name="check-circle" class="size-5" />` inlines the SVG (cached per request) |
| Sizes | 16px in pills and dense tables, 20px default, 24px in navigation and empty states |
| Stroke | 2px at 20 and 24px; 1.75px at 16px |
| Colour | `currentColor` only; the parent sets the colour through a role token |
| Labels | Decorative icons get `aria-hidden="true"`. Icon-only buttons need `aria-label` and a tooltip. |
| RTL | Icons with direction (arrows, chevrons, `log-out`, `undo`, `reply`, `send`) mirror with `rtl:-scale-x-100`. Clocks, checkmarks, media controls and brand marks never mirror. |

Module navigation icons: Dashboard `layout-dashboard`, Students `graduation-cap`, Guardians `users`, Academics `book-open`, Attendance `calendar-check`, Leave `calendar-minus`, Exams `file-text`, Results `award`, Fees `receipt`, Accounts `landmark`, HR `id-card`, Payroll `wallet`, Timetable `table`, Communication `megaphone`, Learning `monitor-play`, Homework `notebook-pen`, Assessment `list-checks`, Library `library`, Transport `bus`, Inventory `boxes`, Discipline `shield-alert`, Reports `bar-chart-3`, Settings `settings`, Approvals `stamp`.

---

## 9.12 Right-to-Left Rules

| Rule | Detail |
|---|---|
| Direction | `<html lang="ur" dir="rtl">` for Urdu UI; `lang="en" dir="ltr"` for English |
| Utilities | Only logical utilities: `ms-*`, `me-*`, `ps-*`, `pe-*`, `start-*`, `end-*`, `text-start`, `text-end`, `rounded-s-*`, `rounded-e-*`, `border-s`, `border-e`. Physical ones (`ml-`, `mr-`, `pl-`, `pr-`, `left-`, `right-`, `text-left`, `text-right`) fail the Blade lint (D3 3.17). |
| Layout | Sidebar sits on the start edge (right in Urdu). Drawers open from the end edge for filters and previews. |
| Tables | Column order stays logical (first column at the start edge). Numbers stay end-aligned. |
| Inputs | Phone, CNIC, email, username, URL, money and numeric fields are always `dir="ltr"` with `text-align: start` in their own direction. Free-text Urdu fields (`name_ur`, remarks) are `dir="rtl" lang="ur"`. |
| Icons | See 9.11 |
| Charts | Axes and legends follow the page direction; time axes still run oldest to newest from the start edge. |
| Dates | Day month year order in both languages: `24 Nov 2026`, `24 نومبر 2026` |

---

## 9.13 Accessibility (WCAG 2.2 AA, FR-GEN-016)

| Criterion | How the system meets it |
|---|---|
| 1.3.1 Info and relationships | Real `<table>`, `<th scope>`, `<label for>`, `<fieldset><legend>` for radio groups and attendance rows |
| 1.4.1 Use of colour | DEC-148: text and icons on every status |
| 1.4.3 Contrast (text) | Verified pairs in 9.4; `DesignTokensTest` fails the build if a token change breaks a pair |
| 1.4.4 / 1.4.10 Resize and reflow | Layouts work at 320px CSS width and 200% zoom; tables become cards |
| 1.4.11 Non-text contrast | `border-strong` on inputs, focus ring 4.46 or more, chart marks with labels where below 3:1 |
| 1.4.12 Text spacing | No fixed-height text containers; Urdu line heights already exceed the criterion |
| 2.1.1 Keyboard | Every Alpine component supports keyboard: arrows in menus, tabs, attendance grid; Esc closes |
| 2.4.3 Focus order | DOM order matches visual order in both directions |
| 2.4.7 / 2.4.11 Focus visible, not obscured | 2px `focus` ring, 2px offset; sticky headers and bars use `scroll-padding` so focused fields are never hidden |
| 2.5.7 Dragging | Timetable and ordering lists that support drag also offer move buttons |
| 2.5.8 Target size | 9.7 targets |
| 3.2.6 Consistent help | "Help" link in the same place in the user menu on every page |
| 3.3.1 / 3.3.3 Error identification and suggestion | Inline errors with icon and fix hint; error summary at the top on submit |
| 3.3.7 Redundant entry | Wizards and multi-step forms carry entered data forward; "same as father" style copy buttons on admission forms |
| 3.3.8 Accessible authentication | Paste allowed in password and OTP fields; OTP fields accept the whole code in one paste; no puzzle captcha (rate limits and honeypot first, an accessible challenge only on abuse) |
| 4.1.3 Status messages | Toasts and bulk progress use `aria-live` (`polite` for success, `assertive` for errors) |

Testing: axe checks run in the browser test suite after UG-1 (D8 8.14) on every key screen in both `en` and `ur`; manual screen reader passes with NVDA (Windows) and TalkBack (Android) before each release.

---

## 9.14 `resources/css/tokens.css`

```css
@theme {
  /* DEC-139: remove Tailwind's default palette */
  --color-*: initial;
  --color-white: #FFFFFF;
  --color-black: #000000;

  /* Primitive scales (use only here, in charts and in branding) */
  --color-primary-50: #E7FDFA;  --color-primary-100: #D1F9F3; --color-primary-200: #A8EFE6;
  --color-primary-300: #72DED2; --color-primary-400: #30C5B8; --color-primary-500: #07A79B;
  --color-primary-600: #088178; --color-primary-700: #067169; --color-primary-800: #005B54;
  --color-primary-900: #044742; --color-primary-950: #002E2A;

  --color-secondary-50: #F4F8FE;  --color-secondary-100: #E7F0FE; --color-secondary-200: #CCE0FE;
  --color-secondary-300: #AECAF4; --color-secondary-400: #8DAEDF; --color-secondary-500: #7092C5;
  --color-secondary-600: #5071A1; --color-secondary-700: #44638F; --color-secondary-800: #354F76;
  --color-secondary-900: #283E5D; --color-secondary-950: #18273D;

  --color-accent-50: #F8F7FE;  --color-accent-100: #F0ECFE; --color-accent-200: #E1D9FE;
  --color-accent-300: #CCBCFE; --color-accent-400: #B498FE; --color-accent-500: #9C70FB;
  --color-accent-600: #7C4CD5; --color-accent-700: #6D3FC0; --color-accent-800: #59309F;
  --color-accent-900: #45247F; --color-accent-950: #2C1455;

  --color-neutral-50: #F6F8FA;  --color-neutral-100: #EDEFF3; --color-neutral-200: #DADFE5;
  --color-neutral-300: #C2C9D1; --color-neutral-400: #A5ADB7; --color-neutral-500: #89919C;
  --color-neutral-600: #69717B; --color-neutral-700: #5B626C; --color-neutral-800: #494F57;
  --color-neutral-900: #393E44; --color-neutral-950: #23272B; --color-neutral-975: #16191C;

  --color-success-50: #EAFEED;  --color-success-100: #D6FADC; --color-success-200: #B2F1BE;
  --color-success-300: #85E099; --color-success-400: #55C875; --color-success-500: #21AC54;
  --color-success-600: #00873C; --color-success-700: #027634; --color-success-800: #015F29;
  --color-success-900: #014B1F; --color-success-950: #013011;

  --color-warning-50: #FEF6ED;  --color-warning-100: #FFECD7; --color-warning-200: #FFD7AA;
  --color-warning-300: #FFB860; --color-warning-400: #EB9812; --color-warning-500: #C67F09;
  --color-warning-600: #9B6200; --color-warning-700: #875505; --color-warning-800: #6D4403;
  --color-warning-900: #563502; --color-warning-950: #382101;

  --color-danger-50: #FFF5F4;  --color-danger-100: #FEEAE8; --color-danger-200: #FFD3CE;
  --color-danger-300: #FEB1AA; --color-danger-400: #FF817A; --color-danger-500: #F24F4E;
  --color-danger-600: #CB242D; --color-danger-700: #B51522; --color-danger-800: #960416;
  --color-danger-900: #77020F; --color-danger-950: #4F0107;

  --color-info-50: #F3F8FF;  --color-info-100: #E5F1FE; --color-info-200: #C9E1FE;
  --color-info-300: #A0CCFE; --color-info-400: #6AB1FE; --color-info-500: #3093F4;
  --color-info-600: #0371CA; --color-info-700: #0162B2; --color-info-800: #014F91;
  --color-info-900: #013E72; --color-info-950: #01274B;

  /* Role tokens: light theme */
  --color-canvas: #F6F8FA;
  --color-surface: #FFFFFF;
  --color-surface-muted: #EDEFF3;
  --color-surface-raised: #FFFFFF;
  --color-border: #DADFE5;
  --color-border-strong: #89919C;
  --color-ink: #23272B;
  --color-ink-muted: #5B626C;
  --color-ink-subtle: #69717B;
  --color-ink-disabled: #A5ADB7;
  --color-ink-inverse: #FFFFFF;
  --color-focus: #088178;
  --color-selection: #D1F9F3;
  --color-overlay: rgb(35 39 43 / 0.55);

  --color-primary: #088178;   --color-primary-strong: #067169;  --color-primary-text: #067169;
  --color-primary-tint: #E7FDFA; --color-primary-on-tint: #005B54; --color-primary-border: #A8EFE6;

  --color-secondary: #283E5D; --color-secondary-strong: #18273D; --color-secondary-text: #44638F;
  --color-secondary-tint: #F4F8FE; --color-secondary-on-tint: #354F76; --color-secondary-border: #CCE0FE;

  --color-accent: #7C4CD5;    --color-accent-strong: #6D3FC0;   --color-accent-text: #6D3FC0;
  --color-accent-tint: #F8F7FE; --color-accent-on-tint: #59309F; --color-accent-border: #E1D9FE;

  --color-success: #00873C;   --color-success-strong: #027634;  --color-success-text: #027634;
  --color-success-tint: #EAFEED; --color-success-on-tint: #015F29; --color-success-border: #B2F1BE;

  --color-warning: #9B6200;   --color-warning-strong: #875505;  --color-warning-text: #875505;
  --color-warning-tint: #FEF6ED; --color-warning-on-tint: #6D4403; --color-warning-border: #FFD7AA;

  --color-danger: #CB242D;    --color-danger-strong: #B51522;   --color-danger-text: #B51522;
  --color-danger-tint: #FFF5F4; --color-danger-on-tint: #960416; --color-danger-border: #FFD3CE;

  --color-info: #0371CA;      --color-info-strong: #0162B2;     --color-info-text: #0162B2;
  --color-info-tint: #F3F8FF; --color-info-on-tint: #014F91;    --color-info-border: #C9E1FE;

  --color-nav-bg: #283E5D;
  --color-nav-ink: #E7F0FE;
  --color-nav-ink-muted: #AECAF4;
  --color-nav-active-bg: #354F76;
  --color-nav-indicator: #B498FE;
  --color-platform-bar: #16191C;

  /* Typography */
  --font-urdu-face: "Darsgah Urdu Nastaliq";
  --font-sans: var(--font-urdu-face), system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans", "Helvetica Neue", Arial, sans-serif;
  --font-numeric: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans", Arial, sans-serif;
  --font-mono: ui-monospace, "SFMono-Regular", Menlo, Consolas, monospace;

  --text-xs: 0.75rem;   --text-xs--line-height: 1rem;
  --text-sm: 0.875rem;  --text-sm--line-height: 1.25rem;
  --text-base: 1rem;    --text-base--line-height: 1.5rem;
  --text-lg: 1.125rem;  --text-lg--line-height: 1.75rem;
  --text-xl: 1.25rem;   --text-xl--line-height: 1.75rem;
  --text-2xl: 1.5rem;   --text-2xl--line-height: 2rem;
  --text-3xl: 1.875rem; --text-3xl--line-height: 2.25rem;
  --text-4xl: 2.25rem;  --text-4xl--line-height: 2.75rem;

  /* Shape, depth, motion */
  --radius-sm: 4px; --radius-md: 6px; --radius-lg: 8px; --radius-xl: 12px;
  --shadow-xs: 0 1px 2px rgb(35 39 43 / 0.06);
  --shadow-sm: 0 1px 3px rgb(35 39 43 / 0.10), 0 1px 2px rgb(35 39 43 / 0.06);
  --shadow-md: 0 4px 8px -2px rgb(35 39 43 / 0.10), 0 2px 4px -2px rgb(35 39 43 / 0.06);
  --shadow-lg: 0 12px 24px -6px rgb(35 39 43 / 0.18);
  --ease-standard: cubic-bezier(0.2, 0, 0, 1);

  /* Layout */
  --container-portal: 40rem;
  --container-form: 48rem;
  --container-app: 90rem;
}

/* Urdu face preference */
[data-urdu-font="naskh"] { --font-urdu-face: "Darsgah Urdu Naskh"; }

/* Urdu text sizing (9.6) */
:lang(ur) { font-size: 1.125em; line-height: 2; letter-spacing: normal; }
:lang(ur) :is(h1, h2, h3, h4) { line-height: 1.8; }

/* Dark theme (R2, DEC-144): role tokens only; primitive steps never change */
[data-theme="dark"] {
  color-scheme: dark;
  --color-canvas: #16191C;
  --color-surface: #23272B;
  --color-surface-muted: #393E44;
  --color-surface-raised: #393E44;
  --color-border: #494F57;
  --color-border-strong: #89919C;
  --color-ink: #F6F8FA;
  --color-ink-muted: #A5ADB7;
  --color-ink-subtle: #89919C;
  --color-ink-disabled: #5B626C;
  --color-ink-inverse: #23272B;
  --color-focus: #72DED2;
  --color-selection: #044742;
  --color-overlay: rgb(0 0 0 / 0.65);

  --color-primary-text: #72DED2; --color-primary-tint: #002E2A; --color-primary-on-tint: #A8EFE6; --color-primary-border: #005B54;
  --color-secondary: #18273D;    --color-secondary-text: #AECAF4; --color-secondary-tint: #18273D; --color-secondary-on-tint: #CCE0FE; --color-secondary-border: #354F76;
  --color-accent-text: #CCBCFE;  --color-accent-tint: #2C1455;  --color-accent-on-tint: #E1D9FE; --color-accent-border: #59309F;
  --color-success-text: #85E099; --color-success-tint: #013011; --color-success-on-tint: #B2F1BE; --color-success-border: #015F29;
  --color-warning-text: #FFB860; --color-warning-tint: #382101; --color-warning-on-tint: #FFD7AA; --color-warning-border: #6D4403;
  --color-danger-text: #FEB1AA;  --color-danger-tint: #4F0107;  --color-danger-on-tint: #FFD3CE; --color-danger-border: #960416;
  --color-info-text: #A0CCFE;    --color-info-tint: #01274B;    --color-info-on-tint: #C9E1FE;  --color-info-border: #014F91;

  --color-nav-bg: #18273D;
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-property: opacity !important;
    transition-duration: 120ms !important;
  }
}
```

Tailwind v4 utilities read tokens through `var(--color-...)`, so redefining a token under `[data-theme="dark"]` or in the tenant branding block (9.20) changes every utility that uses it without rebuilding CSS.

---

## 9.15 Component Library (`x-ui.*`)

### Inventory

| Component | Purpose | Key props | Variants | States |
|---|---|---|---|---|
| `x-ui.button` | Actions | `variant`, `size`, `icon`, `icon-end`, `loading`, `href`, `type` | `primary`, `secondary` (neutral outline), `ghost`, `danger`, `link` | default, hover, active, focus-visible, disabled (with reason), loading |
| `x-ui.input` | Text entry | `name`, `type`, `dir`, `inputmode`, `prefix`, `suffix` | text, number, tel, email, password (reveal toggle), search | default, focus, invalid, read-only, disabled |
| `x-ui.textarea` | Long text | `rows`, `maxlength` (live counter) | | as input |
| `x-ui.select` | Native select, short lists (up to 15) | `options`, `placeholder` | | as input |
| `x-ui.search-select` | Searchable select (Tom Select), long lists, remote search | `endpoint`, `multiple`, `min-chars` | single, multiple | loading, no results |
| `x-ui.checkbox`, `x-ui.radio`, `x-ui.toggle` | Choices | `label`, `description` | | checked, indeterminate (checkbox), disabled |
| `x-ui.field` | Wraps label, control, help, error | `label`, `help`, `required`, `error` | stacked, inline (toggles) | |
| `x-ui.label` | Label with required marker | `for`, `required` | | |
| `x-ui.date-picker` | Date input with calendar | `min`, `max`, `range` | single, range | |
| `x-ui.money` | Display or input money | `amount`, `currency`, `input` | display, input | negative display |
| `x-ui.file-upload` | Upload with type, size and scan status | `accept`, `max`, `multiple` | button, drop zone | uploading (progress), scanning, failed, done |
| `x-ui.table` | Static table | `density` | | |
| `x-ui.data-table` | Filterable, sortable, selectable list | `query`, `columns`, `selectable`, `card-layout` | | loading, empty, filtered-empty, error |
| `x-ui.filter-bar` | Search and filters above tables | `filters`, `saved` | | active filter chips |
| `x-ui.bulk-bar` | Bulk actions for selected rows | `count`, `total`, `actions` | | |
| `x-ui.pagination` | Page navigation | `paginator` | full, simple (mobile) | |
| `x-ui.status-pill` | Mapped status (9.5) | `status`, `enum` | tint, solid (rare) | |
| `x-ui.badge` | Counts, neutral labels | `count`, `tone` | | |
| `x-ui.chip` | Removable filter or tag | `removable` | | |
| `x-ui.avatar` | Photo or initials | `src`, `name`, `size` | 24, 32, 40, 64, 96 px | missing photo (initials) |
| `x-ui.card` | Content container | `title`, `actions`, `padding` | default, muted, interactive | |
| `x-ui.description-list` | Label and value pairs on detail pages | `items`, `columns` | | "Not set" values |
| `x-ui.stat-tile` | KPI number | `label`, `value`, `delta`, `href`, `icon` | default, attention | loading |
| `x-ui.page-header` | Title, breadcrumb, actions | `title`, `subtitle`, `breadcrumbs`, `actions` | | |
| `x-ui.breadcrumb` | Location | `items` | | |
| `x-ui.tabs` | Section switching (server links or in-page) | `items`, `active` | underline, pill | |
| `x-ui.stepper` | Wizard progress | `steps`, `current` | horizontal, vertical (mobile) | done, current, upcoming, error |
| `x-ui.modal` | Dialog | `size`, `title`, `dismissable` | | |
| `x-ui.confirm-dialog` | Confirmation with optional reason or typed check | `level` (1 to 3), `reason`, `confirm-text` | | |
| `x-ui.drawer` | Side panel | `side` (`end` default), `size` | | |
| `x-ui.dropdown` | Menus | `trigger`, `items`, `align` | | |
| `x-ui.tooltip` | Short hint for icon buttons and disabled reasons | `text` | | |
| `x-ui.alert` | Inline message in content | `tone`, `title`, `dismissable` | info, success, warning, danger | |
| `x-ui.banner` | Page-wide notice above content | `tone` | impersonation, all-branches, period-locked, offline, maintenance | |
| `x-ui.toast` | Transient feedback | `tone`, `timeout` | | |
| `x-ui.progress` | Bulk operation progress | `value`, `total`, `label` | bar, inline | running, done, done with errors, failed |
| `x-ui.skeleton` | Loading placeholder | `lines`, `shape` | text, row, tile | |
| `x-ui.loading` | Spinner for buttons and small areas | `size` | | |
| `x-ui.empty-state` | No data | `icon`, `title`, `body`, `action` | empty, filtered, no-permission-scope | |
| `x-ui.icon` | Inline SVG | `name` | | |

### Button

| Variant | Rest | Hover | Text | Border | When |
|---|---|---|---|---|---|
| `primary` | `primary` | `primary-strong` | `ink-inverse` | none | The one main action per screen (UX-02) |
| `secondary` | `surface` | `surface-muted` | `ink` | `border-strong` | Other actions in headers and forms |
| `ghost` | transparent | `surface-muted` | `ink` | none | Toolbars, table row actions, cancel |
| `danger` | `danger` | `danger-strong` | `ink-inverse` | none | Destructive confirmations only (inside a confirm dialog) |
| `link` | transparent | underline | `primary-text` | none | Inline navigation |

| Size | Height | Text | Icon |
|---|---|---|---|
| `sm` | 32px | 14px | 16px |
| `md` (default) | 40px (44px below 768px) | 14px (16px below 768px) | 20px |
| `lg` | 48px | 16px | 20px |

Loading: the label stays, a spinner replaces the icon, the button gets `aria-busy="true"` and ignores further clicks. Every form submit button sets loading on submit, which together with the idempotency key prevents double posting.

```blade
<x-ui.button variant="primary" icon="plus" :href="route('app.student.students.create')">
    {{ __('student::students.add') }}
</x-ui.button>

<x-ui.button variant="secondary" disabled :reason="__('core::locks.period_locked', ['period' => 'Nov 2026'])">
    {{ __('attendance::sheets.edit') }}
</x-ui.button>
```

### Field and input

```
Label text *                         <- 14px, 500, ink; * is aria-hidden, control has aria-required
+---------------------------------+
| 35202-1234567-3                 |  <- 40px (44 mobile), border-strong, radius-md, ink
+---------------------------------+
Enter the 13-digit CNIC with dashes. <- 13px, ink-muted (aria-describedby)
(!) This CNIC is already linked to   <- 13px, danger-text with icon (aria-describedby, aria-invalid)
    guardian Imran Khan.
```

Invalid state: border `danger`, 1px plus an inner 1px ring in `danger`; the error text says what is wrong and how to fix it. Read-only: `surface-muted` background, no border change, still focusable and selectable.

### Data table (desktop)

```
+------------------------------------------------------------------------------------------+
| [ ] | Student             v | Reg no      | Class    | Guardian      | Outstanding | Status   | . |
|-----+-----------------------+-------------+----------+---------------+-------------+----------+---|
| [ ] | (AK) Ayesha Khan      | CS-2024-0153| 9-A      | Imran Khan    |   Rs 3,500  | (!)Partly| : |
|     |      Father: Imran K. |             |          | 0300 1234567  |             |    paid  |   |
| [x] | (BH) Bilal Hussain    | CS-2023-0091| 9-A      | Tariq Hussain |       Rs 0  | (v)Paid  | : |
+------------------------------------------------------------------------------------------+
  Showing 1 to 25 of 312                                     [<] 1 2 3 ... 13 [>]  25 v per page
```

| Rule | Detail |
|---|---|
| Header | `surface-muted`, 13px semibold `ink-muted`, sentence case; sortable headers are buttons with `aria-sort` |
| Rows | 48px comfortable, 36px compact (staff only, never in Urdu); hover `surface-muted`; selected `selection` |
| Alignment | Text at the start; numbers, money and marks at the end with tabular figures; status pills at the start |
| First column | The record's identity (name with avatar, or voucher number); it links to the detail page |
| Actions | Last column: a ghost icon button opening a menu, labelled "Actions for {record}" |
| Sticky | Header sticks under the top bar; the first column sticks on horizontal scroll |
| Selection | Header checkbox selects the page; the bulk bar then offers "Select all 312 matching" |
| Priority | Each column declares `priority` 1 to 3; below `lg` only priority 1 and 2 show; below `md` the table becomes cards |

### Status pill

```
( (v) Paid )     success-tint bg, success-on-tint text, 12px 500, radius-full, 2px 8px padding
( (!) Partly paid )   warning
( (x) Overdue )  danger
```

### Stat tile

```
+---------------------------+
| Today's collection    (i) |  <- 14px ink-muted, icon 20px
| Rs 412,500                |  <- 30px 600 ink, tabular
| 128 receipts  +12% vs avg |  <- 13px; delta uses success-text or danger-text plus arrow icon
+---------------------------+
```

A tile is a link when it has a drill-down. Deltas always have an arrow icon and words, never colour alone.

### Confirm dialog levels

| Level | Use | Content |
|---|---|---|
| 1 | Reversible or low impact (cancel a draft, remove a filter set) | Title, one sentence, Cancel and Confirm |
| 2 | Protected change that the audit needs a reason for (edit finalized attendance, reverse a payment, change a published mark) | What changes (before and after), required reason (10 characters minimum), Cancel and Confirm |
| 3 | High impact or bulk (cancel 214 vouchers, unpublish a result, close a session) | Level 2 plus "Type 214 to confirm" |

```
+------------------------------------------------------+
| Reverse payment R-2026-004512?                    [x]|
|------------------------------------------------------|
| Amount     Rs 5,000                                  |
| Voucher    V-2026-11-000412 (Ayesha Khan, 9-A)       |
| After this, the voucher shows Rs 8,500 outstanding   |
| and the reversal goes to the principal for approval. |
|                                                      |
| Reason *                                             |
| +--------------------------------------------------+ |
| | Cheque bounced, bank memo 22 Nov                 | |
| +--------------------------------------------------+ |
|                                                      |
|                        [ Cancel ]  [ Request reversal ] |  <- danger button
+------------------------------------------------------+
```

### Module composites (examples)

| Component | Module | Shows |
|---|---|---|
| `x-student.identity-chip` | Student | Avatar, name, registration number, class and section; Urdu name below when set |
| `x-attendance.code-cell` | Attendance | One day's code with the 9.5 rendering; keyboard: P, A, L, H keys set codes |
| `x-attendance.legend` | Attendance | All codes with meanings |
| `x-fee.voucher-card` | Fee | Voucher number, month, due date, net, paid, outstanding, status pill, pay or print actions |
| `x-fee.amount-breakdown` | Fee | Charge lines, discounts, late fee, arrears, net |
| `x-result.grade-badge` | Result | Grade with pass or fail meaning (text plus icon) |
| `x-approval.timeline` | Approval | Steps, decisions, actors, timestamps, reasons |
| `x-core.period-lock-notice` | Core | "November 2026 attendance is locked. Request unlock" |

---

## 9.16 Patterns

### Page types

| Type | Structure |
|---|---|
| List | Page header (title, primary action), filter bar, data table, pagination |
| Detail | Page header (identity, status pill, actions menu), summary card, tabs (registered through profile tab registries), activity timeline |
| Form | Page header, error summary (on failed submit), sections in cards, sticky action bar (Cancel, Save) on long forms |
| Wizard | Stepper, one step per page, Back and Continue, data kept between steps, review step before commit |
| Dashboard | Scope line (branch, session, date), "needs attention" list, stat tiles, two or three charts, quick actions |
| Bulk job | Parameters form, preview counts, confirm (level 3), progress page, result summary with downloadable error file |

### Filters and saved views

- Filters live in the URL query string (D3 3.17), so a filtered list can be bookmarked and shared.
- Active filters show as removable chips with a "Clear all" link.
- Saved filters use `saved_report_filters` (D7 7.34) and appear in a "Saved views" dropdown in the filter bar.

### Feedback

| Situation | Pattern |
|---|---|
| Save succeeded | Redirect with a success toast (5 seconds, `aria-live="polite"`) |
| Validation failed | Stay on the form, error summary at the top with links to each field, inline errors, focus moves to the summary |
| Domain rule failed | Danger alert at the top of the form with the translated message and the error code in small text |
| Server error | Error page or inline alert with the correlation id and "Try again" |
| Edit conflict (DEC-136) | Warning banner: "Fatima Ali (Accountant) changed this record at 10:42 AM after you opened it." Actions: "Load latest version" and "Keep my changes and review" (shows the differences) |
| Period locked | Inline `x-core.period-lock-notice` near the action, and the action disabled with the reason |
| Pending approval | Info alert on the record: "Waiting for principal approval since 23 Nov, 3:10 PM" with the approval timeline |
| Autosave grid (attendance, marks) | Small status text: "Saving...", "Saved 10:42 AM", "Not saved, retrying" (with offline banner after 2 failures) |
| Bulk operation | `x-ui.progress` with processed, failed and total counts; leaving the page is safe; an in-app notification arrives when done |

### Empty, loading and error states (FR-GEN-011)

| State | Rendering | Copy pattern |
|---|---|---|
| First use, no records | `x-ui.empty-state` with icon, title, one sentence, and the primary action if the user may create | "No students yet. Add your first student or import a list from Excel." |
| Filtered to nothing | Empty state without illustration, with "Clear filters" | "No vouchers match these filters." |
| Scope empty | Empty state that names the scope | "No classes in Gulberg Branch for session 2026-27." |
| Loading a list | 5 skeleton rows matching column widths | none |
| Loading a dashboard | Skeleton tiles and chart frames; each widget loads on its own | none |
| Error loading part of a page | Inline danger alert inside that card with "Try again"; the rest of the page still works | "Couldn't load today's attendance. Try again. (Ref 01JDA0B1...)" |
| Offline | Warning banner at the top | "You're offline. Changes on this page will be saved when the connection returns." |
| Module not enabled | Not shown at all (navigation hidden, route 404, FR-GEN-012) | none |

### Permissions on screen (DEC-147)

| Case | Rendering |
|---|---|
| No permission for an action | Button or menu item not rendered |
| No permission for a field (masked PII, DEC-62) | Value masked (`35202-•••••••-3`) with a lock icon and tooltip "Full CNIC hidden" |
| Permitted but blocked by state | Disabled control with tooltip and visible short reason |
| Read-only role on an editable page | Page renders in view mode; no edit controls |

### Multi-branch and scope indicators

| Situation | Rendering |
|---|---|
| Organization with one branch | Branch switcher hidden (PR-10) |
| Several branches | Branch switcher in the top bar showing the active branch |
| "All branches" read mode | Info banner: "Viewing all branches. Switch to one branch to make changes." Create and edit actions hidden |
| Session other than the current one | Warning banner: "You are viewing session 2025-26 (closed). Records are read-only." |
| Guardian with several children | Child switcher at the top of every guardian page; the selected child's name and class always visible |

---

## 9.17 Shell Wireframes

### Staff portal, desktop (1280px)

```
+--------------------------------------------------------------------------------------------------+
| [Logo] City School System   Branch: Gulberg v   Session: 2026-27 v      [Search...  /]  [bell 3] [AK v] |  topbar 56px, surface
+-------------------+------------------------------------------------------------------------------+
| nav-bg (264px)    |  Students > All students                                                     |
|                   |  All students                                [Import]  [+ Add student]       |
|  [#] Dashboard    |  --------------------------------------------------------------------------- |
|  [#] Approvals  4 |  [Search name, reg no, guardian mobile ] [Class v] [Section v] [Status v]   |
|  STUDENTS         |  [Filters] (Class 9 x) (Status: Active x)  Clear all          Saved views v  |
| |[#] Students     |  +------------------------------------------------------------------------+ |
|  [#] Admissions   |  | [ ] Student             Reg no       Class  Guardian       Status      : | |
|  [#] Guardians    |  | [ ] (AK) Ayesha Khan    CS-2024-0153 9-A    Imran Khan     (v) Active  : | |
|  ACADEMICS        |  | [ ] (BH) Bilal Hussain  CS-2023-0091 9-A    Tariq Hussain  (v) Active  : | |
|  [#] Classes      |  | ...                                                                    | |
|  [#] Attendance   |  +------------------------------------------------------------------------+ |
|  [#] Exams        |  Showing 1 to 25 of 312                          [<] 1 2 3 ... 13 [>]       |
|  FINANCE          |                                                                              |
|  [#] Fees         |                                                                              |
|  [#] Accounts     |                                                                              |
|  ...              |                                                                              |
|  [<<] Collapse    |                                                                              |
+-------------------+------------------------------------------------------------------------------+
   | = nav-indicator bar (accent-400) on the active item; groups and items come from NavigationRegistry
```

### Staff portal, mobile (360px)

```
+------------------------------------+
| [=]  City School      [bell 3] (AK)|  topbar 56px
| Gulberg v  .  2026-27 v            |  scope line 36px (tap to change)
+------------------------------------+
| All students                       |
| [ + Add student                  ] |  primary full width
| [Search name or reg no      ] [Fil]|
| (Class 9 x) (Active x)             |
| +--------------------------------+ |
| | (AK) Ayesha Khan     (v)Active | |
| | CS-2024-0153  .  9-A           | |
| | Guardian: Imran Khan        :  | |
| +--------------------------------+ |
| +--------------------------------+ |
| | (BH) Bilal Hussain   (v)Active | |
| | CS-2023-0091  .  9-A           | |
| | Guardian: Tariq Hussain     :  | |
| +--------------------------------+ |
| [ Load more (25 of 312) ]          |
+------------------------------------+
```

### Guardian portal, mobile (360px)

```
+------------------------------------+
| [Logo] City School       [bell 2]  |
| ( Ayesha Khan, 9-A        v )      |  child switcher, always visible
+------------------------------------+
| Today                              |
| +--------------------------------+ |
| | (v) Present, arrived 7:52 AM   | |
| +--------------------------------+ |
| +--------------------------------+ |
| | Fee due 10 Nov        Rs 3,500 | |
| | (!) Partly paid                | |
| | [ Pay now ]   [ View voucher ] | |
| +--------------------------------+ |
| Diary for today                    |
| +--------------------------------+ |
| | Maths: Exercise 4.2, Q1 to 10  | |
| | Urdu: نظم یاد کریں              | |
| +--------------------------------+ |
| Announcements (1 new)          >   |
+------------------------------------+
| [Home] [Attend.] [Fees] [Results] [More] |  bottom nav 64px, labels always shown
+------------------------------------+
```

### Student portal, mobile (360px)

```
+------------------------------------+
| [Logo] City School   [bell] (AK)   |
+------------------------------------+
| Good morning, Ayesha               |
| Next: Physics, 9:40 AM, Room 12    |
| Due soon                           |
|  - Chemistry worksheet, tomorrow   |
|  - Maths quiz 3, opens 2:00 PM     |
| Continue learning                  |
|  [ Physics: Motion, lecture 4  > ] |
+------------------------------------+
| [Home] [Timetable] [Homework] [Learn] [More] |
+------------------------------------+
```

### Platform portal

```
+--------------------------------------------------------------------------------------+
| (PLATFORM)  Darsgah Platform        [Search organizations...]         [SA Admin v]    |  platform-bar #16191C, badge warning-tint
+--------------+-----------------------------------------------------------------------+
| Organizations|  Organizations                                         [+ Provision]  |
| Plans        |  ...                                                                  |
| Support      |                                                                       |
| Announcements|                                                                       |
| Audit        |                                                                       |
+--------------+-----------------------------------------------------------------------+
```

### Impersonation banner (support access, D5)

```
+--------------------------------------------------------------------------------------+
| (!) Support session: you are viewing City School as Fatima Ali (Accountant).         |
|     Access granted by Owner until 3:00 PM. Every action is logged.   [ End session ] |
+--------------------------------------------------------------------------------------+
   warning-tint background, warning-on-tint text, z-topbar, cannot be dismissed
```

### Sign-in page

```
+------------------------------------------------+
|              [ Organization logo ]             |   accent band 8px across the top (tenant colour)
|                City School System              |
|        English | اردو                           |
|  +------------------------------------------+  |
|  | Mobile number, email or username         |  |
|  | [ 0300 1234567                         ] |  |
|  | Password                       Show      |  |
|  | [ ********                             ] |  |
|  | [ Sign in                              ] |  |   primary, full width
|  | Forgot password?   First time? Get code  |  |
|  +------------------------------------------+  |
|      Powered by Darsgah  .  Help  .  Privacy   |
+------------------------------------------------+
```

D10 contains the full screen inventory and the wireframes for each key screen built from these shells.

---

## 9.18 Data Visualization

### Categorical palette (DEC-149)

Fixed order; never cycled. Validated on this system's surfaces (`#FFFFFF` light, `#23272B` dark): every adjacent pair passes colour-vision-deficiency separation (worst 9.1 light, 8.4 dark, target 8 or more) and normal-vision separation (worst 19.6 light, 19.3 dark, floor 15).

| Slot | Hue | Light | Dark |
|---|---|---|---|
| 1 | blue | `#2A78D6` | `#3987E5` |
| 2 | orange | `#EB6834` | `#D95926` |
| 3 | aqua | `#1BAF7A` | `#199E70` |
| 4 | yellow | `#EDA100` | `#C98500` |
| 5 | magenta | `#E87BA4` | `#D55181` |
| 6 | green | `#008300` | `#0CA30C` |
| 7 | violet | `#6250D6` | `#9085E9` |
| 8 | red | `#E34948` | `#E66767` |

| Rule | Detail |
|---|---|
| Light-mode contrast | Aqua (2.82), yellow (2.17) and magenta (2.69) are below 3:1 on white, so charts using them show direct labels or offer the table view |
| Series count | Line, bar and stacked charts: up to 8; a ninth series folds into "Other". Scatter and bubble charts: first 3 slots only (validated for all pairs) |
| Same entity, same colour | Colour follows the entity (for example each branch keeps its slot across every chart on the page), never its rank |
| Status | Charts showing good, warning or bad use the role tokens `success`, `warning`, `danger` with icons and labels, never a series slot |
| Sequential | One hue, primary teal, light to dark: steps 100 to 900 for heatmaps (attendance rate by section and day); ordinal scales start at step 400 |
| Diverging | `danger` arm to `primary` arm with `neutral-100` (`#EDEFF3`) as the midpoint (for example attendance against target) |
| Axes | One y-axis only, never a dual axis; gridlines `border`; axis labels `ink-muted` 12px |
| Marks | Lines 2px, markers 8px, bars with 4px rounded data ends and a 2px gap |
| Labels | Legend always present for 2 or more series; direct labels on up to 4 series; values in `ink`, never in the series colour |
| Interaction | Hover or tap tooltip on every mark; crosshair on line charts; every chart has a "View as table" toggle |
| Texture | Print and forced-colours mode switch fills to 45 and 135 degree line textures |

The Chart.js theme (`resources/js/charts/theme.js`) reads these values from CSS custom properties, so dark mode and printing pick the right steps automatically.

---

## 9.19 Print and PDF

| Rule | Detail |
|---|---|
| Engine | Gotenberg (DEC-31) from Blade templates in `{module}::pdf.*`, styled by `pdf.css` |
| Fonts | Embedded: Noto Sans (Latin), Noto Nastaliq Urdu, Noto Naskh Arabic, installed in the Gotenberg image |
| Colour | Black text on white; the header band uses `secondary` or the tenant brand colour; no status colour carries meaning (DEC-148) |
| Margins | A4 and Legal: 12mm; CR80 cards: 3mm safe area |
| Header | Logo (18mm high), institution name in English and Urdu, branch address and phone |
| Footer | Printed by, printed at (organization timezone), verification code with QR (Document module), page x of y |
| Numbers | Tabular figures; amounts also in words on vouchers and receipts |

| Document | Page | Layout |
|---|---|---|
| Fee voucher | A4 landscape | Three copies side by side (Bank, School, Parent), dashed cut lines, barcode and consumer number on each copy |
| Fee receipt | A5 portrait (2 per A4) or 80mm thermal | Receipt number, payer, voucher, allocations, method, cashier |
| Result card | A4 portrait | Header, student identity with photo, subject table, totals, grade, position (if enabled), attendance, remarks, signatures |
| Admit card | A5 portrait (2 per A4) | Photo, roll number, date sheet, instructions |
| ID card | CR80 (85.6 x 54 mm), 10 per A4 sheet | Photo, name, class, registration number, QR, validity, tenant colour band |
| Certificates | A4 portrait | Formal layout with serial number, verification QR and signature images |
| Payslip (R2) | A4 portrait | Earnings, deductions, net, attendance summary |

`print.css` for browser printing hides navigation, bars and buttons, expands tables to full width, and repeats table headers on each page.

---

## 9.20 Tenant Branding (DEC-141)

| Input (Settings) | Rules |
|---|---|
| Logo | SVG or PNG, at least 512px wide, transparent background preferred; resized copies for top bar (32px high), sign-in (64px), PDF (18mm), ID card |
| Brand colour | One HEX value |
| Document headers and signatures | Per institution (D2 Document module) |

**Generating the accent scale**

1. Convert the brand colour to OKLCH (lightness, chroma, hue).
2. Build steps 50 to 950 at the system's fixed lightness targets (0.978, 0.952, 0.902, 0.832, 0.745, 0.655, 0.545, 0.495, 0.425, 0.36, 0.27) using the brand's hue and chroma, reduced at the light and dark ends and clamped to the sRGB gamut.
3. Check the gates below; if a gate fails, move that step's lightness in 0.01 steps (darker for text gates, lighter for the navigation gate) until it passes.
4. Save the 11 values with the organization's settings; the layout writes them into a nonce-tagged `<style>` block as `--color-accent-*` and the accent role tokens.

| Gate | Requirement |
|---|---|
| G1 | White text on `accent-600` at least 4.5:1 (brand band, sign-in band text) |
| G2 | `accent-700` on white at least 4.5:1 (accent text) |
| G3 | `accent-400` on `nav-bg` at least 3:1 (active navigation indicator) |
| G4 | `ink` on `accent-50` at least 12:1 (tinted brand surfaces) |

**Examples (computed):**

| Brand colour | accent-400 | accent-600 | accent-700 | G1 | G2 | G3 |
|---|---|---|---|---|---|---|
| Maroon `#7A1F2B` | `#EC8D91` | `#AD4E54` | `#994148` | 5.26 | 6.55 | 4.53 |
| Green `#1B5E20` | `#81BE81` | `#408142` | `#357237` | 4.73 | 5.81 | 4.96 |
| Gold `#F4B400` | `#DBA105` | `#906900` | `#7E5B00` | 4.99 | 6.21 | 4.70 |

The settings screen shows a live preview (top bar, sign-in page, voucher header, ID card) and a warning, not a block, when the hue is within 20 degrees of the danger hue: "This colour is close to the error colour. It's allowed, because errors always show an icon and text."

Where the accent appears: the sign-in band; the navigation indicator; brand band on PDFs and ID cards; guardian and student portal headers; selected child chip. Where it never appears: primary buttons, links, status pills, chart series.

---

## 9.21 Content and Formatting

### Voice

| Rule | Example |
|---|---|
| Plain words, short sentences | "Voucher cancelled." not "The voucher has been successfully cancelled." |
| Sentence case everywhere | "Record payment", "Fee structures" |
| Buttons start with a verb | "Generate vouchers", "Mark attendance", "Request unlock" |
| Errors say what happened and what to do | "This voucher is already paid. Open the receipt to print it again." |
| No blame, no jargon | "We couldn't reach the SMS provider. The message will retry in 5 minutes." |
| Name people and times in history | "Changed by Fatima Ali, 23 Nov 2026, 3:10 PM" |

### Formats

| Item | English | Urdu UI |
|---|---|---|
| Date | `24 Nov 2026` | `24 نومبر 2026` (Western digits) |
| Compact date (dense tables) | `24/11/2026` | `24/11/2026` |
| Time | `9:05 AM` | `9:05` with the Urdu meridiem label from the date library |
| Date and time | `24 Nov 2026, 9:05 AM` | same order |
| Money | `Rs 8,500`; paisa shown only when not zero: `Rs 8,500.50` | same, isolated LTR |
| Negative money | `-Rs 500`; accounting statements use `(Rs 500)` | same |
| Number grouping | International (`850,000`) by default; South Asian (`8,50,000`) as an organization setting `localization.number_grouping` | same |
| Phone | `0300 1234567` (from E.164) | same, LTR |
| CNIC | `35202-1234567-3`, masked `35202-•••••••-3` without `core.pii.view_full` | same, LTR |
| Percentages | `92.5%` | same |
| Empty value | "Not set" in `ink-subtle` | "درج نہیں" |
| Names | "Ayesha Khan", second line "Father: Imran Khan" | Urdu name when set, else English |

### Urdu terminology (defaults; organizations can override through `translation_overrides`)

| English | Urdu |
|---|---|
| Student | طالب علم |
| Guardian | سرپرست |
| Class | جماعت |
| Section | سیکشن |
| Attendance | حاضری |
| Leave | رخصت |
| Fee voucher | فیس واؤچر |
| Exam | امتحان |
| Result | نتیجہ |
| Homework | ہوم ورک |
| Admission | داخلہ |
| Branch | برانچ |
| Principal | پرنسپل |
| Announcement | اعلان |

---

## 9.22 Governance

| Rule | Mechanism |
|---|---|
| One source of tokens | `resources/css/tokens.css`; HEX values appear nowhere else in views, CSS or JS except the chart theme |
| No raw palette | DEC-139 removes default colours; `ForbiddenCodeTest` (D8 8.14) also fails on `#` HEX literals and `style="color` in Blade files |
| No physical direction utilities | `BladeConventionsTest` (a new arch test next to `RouteConventionsTest`) fails on `ml-`, `mr-`, `pl-`, `pr-`, `left-`, `right-`, `text-left`, `text-right` |
| Contrast never regresses | `tests/Unit/Core/DesignTokensTest.php` parses `tokens.css` and asserts every pair in 9.4 (light and dark) and the navigation pairs |
| Branding gates tested | Unit tests for the generator with edge colours (pure yellow, near-black, grey, neon green) |
| New component | Added to `components/ui` only when used by two or more modules; otherwise it stays a module composite |
| Visual regression | After UG-1, Pest 4 browser tests capture key screens in `en`/`ur` and light/dark |
| Changes | A token change updates this document, `tokens.css`, the test pairs and the screenshot baselines in one change |

---

## Assumptions

| ID | Assumption |
|---|---|
| A-74 | The palette is the product default until a brand identity is commissioned; a rebrand changes values in `tokens.css`, not token names. |
| A-75 | System fonts give acceptable Latin rendering on Windows, Android, iOS and macOS; no Latin web font is needed. |
| A-76 | Lucide's ISC licence allows copying the SVG subset into the repository with the licence file. |
| A-77 | Chrome on budget Android phones shapes Nastaliq correctly; the Naskh option covers slow devices and personal preference. |
| A-78 | Dark theme waits for R2 so R1 testing covers one theme in two languages. |

## Gaps Found and Added

| ID | Gap | Decision |
|---|---|---|
| GAP-186 | Tenant colours could break contrast or change status meaning | Accent-only branding with four gates (DEC-141, 9.20) |
| GAP-187 | Tailwind's built-in palette would let views bypass tokens | Palette reset (DEC-139) |
| GAP-188 | A Nastaliq web font is heavy for low-bandwidth users | `unicode-range` loading and the Naskh option (DEC-142) |
| GAP-189 | Numbers and codes garble inside RTL text | LTR isolation rule (DEC-143) |
| GAP-190 | D8 component list lacked banner, chip, description list, progress, stepper, tooltip, filter bar and bulk bar | Added (9.15) |
| GAP-191 | No rule on hiding versus disabling actions | DEC-147 |
| GAP-192 | Light input borders would fail WCAG 1.4.11 | `border-strong` at 3.18:1 |
| GAP-193 | Chart colours were undefined | Validated palette (DEC-149, 9.18) |
| GAP-194 | Platform staff could mistake the platform portal for a tenant | Distinct platform bar and impersonation banner (DEC-150) |
| GAP-195 | Destructive confirmation strength was undefined | Three confirm levels (9.15) |
| GAP-196 | South Asian digit grouping preference not covered | `localization.number_grouping` setting (9.21) |
| GAP-197 | Token edits could silently break contrast | `DesignTokensTest` (9.22) |
| GAP-198 | PDF fonts were not specified | Fonts inside the Gotenberg image (9.19) |

---

Reply CONTINUE for Deliverable 10
