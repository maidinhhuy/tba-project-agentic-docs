---
name: TBA-agentic
description: Prototype-as-a-service platform. shadcn/ui on Next.js + Tailwind; this DESIGN.md specifies the brand-layer delta only. All unlisted tokens inherit shadcn defaults.
status: final
sources:
  - _bmad-output/planning-artifacts/briefs/brief-AI-driven-2026-06-11/brief.md
created: 2026-06-11
updated: 2026-06-20
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
    # Placement: project header area alongside status-badge (not below milestone list)
  milestone-stepper:
    circleSize-default: '48px'
    circleSize-active: '56px'
    circleDone: 'bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)]'
    circleActive: 'bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)]'
    circlePending: 'bg-background border-2 border-border text-muted-foreground'
    connectorHeight: '2px'
    connectorDone: 'bg-primary'
    connectorPending: 'bg-border'
    expandPanel: 'bg-teal-50 border border-teal-100 rounded-lg'
    # expand-panel arrow: pseudo-element triangle pointing up, teal-50 fill
  delivery-modal:
    width: 'max-w-[500px]'
    radius: 'rounded-[14px]'
    deliverableGroup: 'bg-muted border border-border rounded-lg overflow-hidden'
    orDivider: 'text-muted-foreground text-xs font-bold tracking-wide uppercase'
    fileZone: 'border-dashed border-2 border-border rounded-md hover:border-primary hover:bg-teal-50 transition-colors'
    # File zone: dashed border, centered icon + copy, limit badge
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

- **`revision-counter`** — Inline chip in the Project Detail **header** alongside the status badge (not below the milestone stepper). Format: `"Còn X lần chỉnh sửa"`. Color is `text-muted-foreground` by default; switches to `text-destructive` and text becomes `"Còn 1 lần chỉnh sửa cuối"` when X ≤ 1. Never shown on the project list card.

- **`milestone-stepper`** — Full-width horizontal stepper on Customer Project Detail, inside the project header card. Three steps connected by a horizontal line. Done circles: 48px teal filled + checkmark icon, teal shadow. Active circle: 56px teal filled, `margin-top: -4px` to break out of the line slightly. Pending circles: 48px, white bg, 2px border, muted text. Connector line: 2px; fills teal for completed segments, `border` color for pending. When a step is active: an expand panel (`{components.milestone-stepper.expandPanel}`) renders below the full stepper row, pointing up via a pseudo-element triangle. Panel contains deliverable link, delivery note, and "Chấp nhận" / "Yêu cầu chỉnh sửa" action buttons.

- **`delivery-modal`** — shadcn `Dialog`, `{components.delivery-modal.width}`, `{components.delivery-modal.radius}`. Header: small teal eyebrow label + milestone name as `text-lg font-bold` title + close icon. Body: **Deliverable group** (`{components.delivery-modal.deliverableGroup}`) — URL input row (icon prefix + full-width input) stacked above file dropzone (`{components.delivery-modal.fileZone}`), separated by an "Hoặc" (`{components.delivery-modal.orDivider}`) divider; group header label includes a muted "Bắt buộc có ít nhất 1" tag. Below the group: optional `Textarea` for delivery note with `text-xs text-muted-foreground` hint. Footer: left-aligned informational note (info icon + text) + right-aligned Cancel + "Giao ngay →" primary button.

## Do's and Don'ts

| Do | Don't |
|---|---|
| Use `{colors.primary}` for primary buttons and active nav only | Use teal for decorative elements or illustrations |
| Status badges always: icon + color + text label | Use status colors without accompanying text (color-only = inaccessible) |
| `project-card` hover with `border-teal-300 shadow-sm` | Add elevation or glow effects beyond the defined hover state |
| `revision-counter` only on Project Detail | Show revision counter on the project list card |
| Inherit shadcn defaults for all unlisted components | Override shadcn tokens not listed in this DESIGN.md |
| Use `rounded-full` only for badge and counter chips | Apply pill shapes to cards, buttons, or form elements |
