---
name: TBA-agentic
description: Prototype-as-a-service platform. shadcn/ui on Next.js + Tailwind; this DESIGN.md specifies the brand-layer delta only. All unlisted tokens inherit shadcn defaults.
status: final
sources:
  - _bmad-output/planning-artifacts/briefs/brief-AI-driven-2026-06-11/brief.md
created: 2026-06-11
updated: 2026-06-11
colors:
  primary: '#0D9488'
  primary-foreground: '#FFFFFF'
  # Status semantic tokens — used exclusively by status-badge component
  status-submitted: '#EFF6FF'
  status-submitted-text: '#1E40AF'
  status-analyzing: '#F0FDFA'
  status-analyzing-text: '#0F766E'
  status-in-development: '#CCFBF1'
  status-in-development-text: '#0D9488'
  status-awaiting-review: '#FFFBEB'
  status-awaiting-review-text: '#92400E'
  status-in-revision: '#FFF7ED'
  status-in-revision-text: '#9A3412'
  status-finalizing: '#ECFDF5'
  status-finalizing-text: '#065F46'
  status-delivered: '#F0FDF4'
  status-delivered-text: '#166534'
  status-cancelled: '#FEF2F2'
  status-cancelled-text: '#991B1B'
typography:
  # Geist Sans inherited from shadcn — no override needed.
  # Project names rendered at body-lg weight semibold, not a custom typeface.
rounded:
  # shadcn defaults inherited as-is.
spacing:
  # shadcn / Tailwind 4-base scale inherited. Max content width: max-w-5xl.
components:
  button-primary:
    background: '{colors.primary}'
    foreground: '{colors.primary-foreground}'
    radius: 'rounded-md'
  project-card:
    border: '1px solid hsl(var(--border))'
    radius: 'rounded-lg'
    padding: '24px'
    hover: 'border-teal-300 shadow-sm transition-shadow'
  status-badge:
    radius: 'rounded-full'
    padding: '2px 10px'
    fontSize: '12px'
    fontWeight: '500'
    # Background and text color are driven by semantic token pair per state (see Colors)
    # Always render icon + color + text — never color alone (accessibility)
  revision-counter:
    fontSize: '13px'
    defaultColor: 'text-muted-foreground'
    warningColor: 'text-destructive'
    # Warning threshold: ≤ 1 revision remaining
---

## Brand & Style

TBA-agentic delivers working software from ideas — fast, structured, and extensible. The visual identity follows the product promise: **clarity over decoration**. Every surface should communicate progress and control at a glance, without visual noise.

TBA-agentic inherits shadcn/ui defaults wholesale. This DESIGN.md specifies only the brand-layer delta: primary teal, semantic status color system, and two custom components (project card, status badge). Everything not listed here — Button variants, Dialog, Sheet, Toast, Tabs, Avatar, Separator, Input — uses shadcn as-is. Overriding those without justification is against brand discipline.

The tone is **professional-minimal**: confident like a SaaS tool, approachable enough for a first-time founder. Not corporate, not startup-hype.

## Colors

TBA-agentic uses one brand color and a semantic system for project status. Nothing else.

- **Teal Primary (`#0D9488`)** — the single brand color. Used on primary buttons, active nav items, and the `IN_DEVELOPMENT` status badge. Replaces shadcn's default `primary`.
- **Status semantic palette** — eight token pairs (background + text), one per project state. These tokens exist only to power the `status-badge` component. They are never used decoratively, never used for chrome.
- **All other tokens** (`background`, `foreground`, `muted`, `muted-foreground`, `border`, `input`, `ring`, `card`, `popover`, `destructive`) inherit shadcn defaults without override.

**Status color map:**

| State | Badge background | Badge text | Meaning signal |
|---|---|---|---|
| SUBMITTED | `{colors.status-submitted}` | `{colors.status-submitted-text}` | Received, waiting |
| ANALYZING | `{colors.status-analyzing}` | `{colors.status-analyzing-text}` | Admin reviewing |
| IN_DEVELOPMENT | `{colors.status-in-development}` | `{colors.status-in-development-text}` | Active, building |
| AWAITING_REVIEW | `{colors.status-awaiting-review}` | `{colors.status-awaiting-review-text}` | Attention needed |
| IN_REVISION | `{colors.status-in-revision}` | `{colors.status-in-revision-text}` | Revision in flight |
| FINALIZING | `{colors.status-finalizing}` | `{colors.status-finalizing-text}` | Nearly done |
| DELIVERED | `{colors.status-delivered}` | `{colors.status-delivered-text}` | Complete |
| CANCELLED | `{colors.status-cancelled}` | `{colors.status-cancelled-text}` | Ended |

Avoid: using status colors outside the badge component; using `{colors.primary}` for status badges; inventing new semantic colors without a corresponding state.

## Typography

Geist Sans (shadcn default) across all surfaces — no override. The hierarchy is achieved through weight and size, not typeface switching.

- **Project name on card:** `text-base font-semibold` — prominent without a custom display font
- **Section headers:** `text-sm font-medium text-muted-foreground uppercase tracking-wide`
- **Body / descriptions:** `text-sm text-foreground`
- **Timestamps, metadata:** `text-xs text-muted-foreground`

No serif moments. TBA-agentic is a tool, not an editorial product.

## Layout & Spacing

shadcn / Tailwind 4-base spacing scale inherited. Two layout contexts:

- **Customer portal:** `max-w-4xl` centered, single-column card grid. Cards in 1-column on `sm`, 2-column on `md+`.
- **Admin dashboard:** `max-w-6xl`, full-width table layout. Sidebar navigation on `lg+`.

Both contexts: `px-4` on `sm`, `px-6` on `md`, `px-8` on `lg+`.

## Elevation & Depth

Inherited from shadcn. One brand addition: `project-card` gains `shadow-sm` on hover (via `{components.project-card.hover}`) to signal interactivity without a heavy shadow system.

## Shapes

shadcn defaults inherited (`rounded-md` for buttons and inputs, `rounded-lg` for cards and dialogs). `rounded-full` used only for `status-badge` and `revision-counter` chip. No pill shapes elsewhere.

## Components

TBA-agentic uses the following shadcn components as-is, unchanged: `Button`, `Card`, `Dialog`, `Sheet`, `Input`, `Textarea`, `Select`, `Toast`, `Table`, `Badge`, `Separator`, `Avatar`, `Tabs`. Do not override these.

Brand-layer components:

- **`project-card`** — Customer-facing card per project. Layout: project name (semibold) + status badge on the same row, milestone subtitle below, timestamp + "View details" link at bottom. Border on default state; `border-teal-300 shadow-sm` on hover. Uses shadcn `Card` as base structure.

- **`status-badge`** — Always renders three elements: a filled circle icon (4px, same color as badge text), the state label, and the background pill. Never render color alone — always include the text label for accessibility. Color pair driven by semantic token map above.

- **`revision-counter`** — Inline chip on Project Detail surface only. Format: `"X revisions remaining"`. Color is `text-muted-foreground` by default; switches to `text-destructive` when X ≤ 1. Never shown on the project list card.

## Do's and Don'ts

| Do | Don't |
|---|---|
| Use `{colors.primary}` for primary buttons and active nav only | Use teal for decorative elements or illustrations |
| Status badges always: icon + color + text label | Use status colors without accompanying text (color-only = inaccessible) |
| `project-card` hover with `border-teal-300 shadow-sm` | Add elevation or glow effects beyond the defined hover state |
| `revision-counter` only on Project Detail | Show revision counter on the project list card |
| Inherit shadcn defaults for all unlisted components | Override shadcn tokens not listed in this DESIGN.md |
| Use `rounded-full` only for badge and counter chips | Apply pill shapes to cards, buttons, or form elements |
