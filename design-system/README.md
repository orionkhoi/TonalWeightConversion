# TonalPlan — Design System

The visual building blocks behind the TonalPlan app, extracted so they can be reused,
handed to designers, or synced into a Claude Design project.

**Dark, high-contrast, one lime accent, generous rounding, calm hairline borders.**

## Contents

| File | What it is |
|---|---|
| [`index.html`](index.html) | **Full showcase** — open this first. Every foundation and component on one page. |
| [`tokens.css`](tokens.css) | Design tokens (colors, type, radius, spacing) as CSS variables. |
| `foundations/colors.html` | Color palette swatches. |
| `foundations/typography.html` | Type scale (logo, h1, h2, label, body, small). |
| `components/buttons.html` | Primary + ghost buttons, small + disabled. |
| `components/inputs.html` | Text / number inputs and selects, incl. focus state. |
| `components/tabs.html` | Pill tab bar. |
| `components/badges.html` | Metadata badges + the live sync-status dot. |
| `components/cards.html` | Result readout card + list rows. |

## How to view

Open `index.html` in any browser, or from the repo root serve it and visit `/design-system/`.

## Design-sync ready

Each file in `foundations/` and `components/` starts with a `<!-- @dsCard group="…" name="…" -->`
marker, so if this folder is pushed to a Claude Design project it renders as a labelled card in
the Design System pane. (Direct sync needs an interactive login that the web session doesn't have —
seed the project with Claude Design's "Send to Claude Code Web," or bring these files in manually.)

## Tokens at a glance

```
Background   #0F0F0F      Accent       #C8FF00
Surface      #1A1A1A      Text         #F0F0F0
Surface inset#111111      Muted        #8A8A8A
Border       #2A2A2A      Warn/saving  #F5B301   Danger #FF4444

Radius       6px badges · 10px buttons/inputs · 16px cards
Font         -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif
```
