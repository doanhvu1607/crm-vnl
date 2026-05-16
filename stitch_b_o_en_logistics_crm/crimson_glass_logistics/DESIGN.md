---
name: Crimson Glass Logistics
colors:
  surface: '#f8f9fa'
  surface-dim: '#d9dadb'
  surface-bright: '#f8f9fa'
  surface-container-lowest: '#ffffff'
  surface-container-low: '#f3f4f5'
  surface-container: '#edeeef'
  surface-container-high: '#e7e8e9'
  surface-container-highest: '#e1e3e4'
  on-surface: '#191c1d'
  on-surface-variant: '#5d3f3d'
  inverse-surface: '#2e3132'
  inverse-on-surface: '#f0f1f2'
  outline: '#926e6c'
  outline-variant: '#e7bcba'
  surface-tint: '#bf0022'
  primary: '#ac001e'
  on-primary: '#ffffff'
  primary-container: '#d90429'
  on-primary-container: '#ffeae8'
  inverse-primary: '#ffb3af'
  secondary: '#5b5d74'
  on-secondary: '#ffffff'
  secondary-container: '#ddddf9'
  on-secondary-container: '#5f6178'
  tertiary: '#495567'
  on-tertiary: '#ffffff'
  tertiary-container: '#616d81'
  on-tertiary-container: '#e8efff'
  error: '#ba1a1a'
  on-error: '#ffffff'
  error-container: '#ffdad6'
  on-error-container: '#93000a'
  primary-fixed: '#ffdad7'
  primary-fixed-dim: '#ffb3af'
  on-primary-fixed: '#410005'
  on-primary-fixed-variant: '#930018'
  secondary-fixed: '#e0e0fc'
  secondary-fixed-dim: '#c4c4df'
  on-secondary-fixed: '#181a2e'
  on-secondary-fixed-variant: '#43455b'
  tertiary-fixed: '#d7e3fa'
  tertiary-fixed-dim: '#bbc7dd'
  on-tertiary-fixed: '#101c2c'
  on-tertiary-fixed-variant: '#3c475a'
  background: '#f8f9fa'
  on-background: '#191c1d'
  surface-variant: '#e1e3e4'
typography:
  headline-lg:
    fontFamily: Inter
    fontSize: 28px
    fontWeight: '300'
    lineHeight: 36px
    letterSpacing: -0.02em
  headline-md:
    fontFamily: Inter
    fontSize: 22px
    fontWeight: '300'
    lineHeight: 28px
  body-md:
    fontFamily: Inter
    fontSize: 14px
    fontWeight: '300'
    lineHeight: 22.4px
  body-sm:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: '300'
    lineHeight: 19.2px
  label-md:
    fontFamily: Inter
    fontSize: 13px
    fontWeight: '500'
    lineHeight: 16px
    letterSpacing: 0.05em
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  sidebar-width: 280px
  container-padding: 32px
  gutter: 24px
  stack-sm: 8px
  stack-md: 16px
  stack-lg: 24px
---

## Brand & Style
The design system focuses on high-precision logistics management for Báo Đen Logistics. The brand personality is efficient, authoritative, and technologically advanced. 

The aesthetic combines **Modern Minimalism** with **Glassmorphism**. It utilizes a vast "white-space" philosophy to reduce cognitive load in data-heavy CRM environments. Visual depth is achieved through translucent glass overlays and subtle backdrop blurs, creating a sense of lightness and sophisticated layering without sacrificing professional clarity.

## Colors
The palette is rooted in a clean, high-contrast foundation.

- **Primary Accent (#D90429):** Used for critical actions, active states, and brand identifiers. It provides high visibility against the neutral background.
- **Headings (#2B2D42):** A deep charcoal used for text hierarchy to ensure maximum readability and a grounded feel.
- **Body & Secondary Text (#8D99AE):** A soft slate gray that maintains legibility while reducing visual noise for long-form content.
- **Backgrounds:** The primary workspace uses Pure White (#FFFFFF), while secondary surfaces and containers utilize a very light gray (#F8F9FA) to define boundaries.

## Typography
The system exclusively uses **Inter** set at a **Light (300)** weight to evoke a modern, technical, and airy feeling. 

- **Headlines:** Use a refined 28px size with tight letter-spacing to maintain a professional editorial look.
- **Body Text:** Set at 14px with a generous 1.6 line-height to ensure that dense logistics data and shipment details are easily scannable and comfortable to read.
- **Labels:** Utilize a slightly heavier weight (500) and uppercase styling at smaller sizes for metadata and table headers to provide contrast against the predominantly light-weight body text.

## Layout & Spacing
The layout follows a **Fixed Sidebar** model with a fluid content area. 

- **Sidebar:** A rigid 280px white sidebar on the left. Active navigation states are marked by a vertical Crimson Red line on the far left edge of the menu item.
- **Grid:** Content is organized into a flexible 12-column system for the main dashboard, using 24px gutters.
- **Rhythm:** Spacing is high to reflect the "Minimalist" style. Content blocks should have ample breathing room (32px padding) to separate different data modules.

## Elevation & Depth
Depth is created through two primary methods:

1.  **Glassmorphism:** Overlays use a 70% white opacity with a **15px to 20px backdrop-blur**. This is applied to modal windows, dropdown menus, and floating navigation bars.
2.  **Soft Shadows:** Elevated elements like cards use a very soft, diffused shadow (0px 10px 30px rgba(43, 45, 66, 0.05)) to lift them slightly off the #F8F9FA background tiers.
3.  **Borders:** Physical separation is reinforced by 1px light gray borders (#EDF2F4) on card elements.

## Shapes
The design system utilizes a varying radius strategy to distinguish between structural components and interactive elements:

- **Primary Container Radius:** Large components like cards and main dashboard sections use a **24px radius** for a soft, modern appearance.
- **Interactive Radius:** Buttons and input fields use a tighter **12px radius** to feel more precise and responsive.
- **Utility Radius:** Badges and chips are fully pill-shaped (100px) to differentiate them from functional inputs.

## Components

- **Cards:** 24px corner radius, 1px light gray border, and a soft shadow. Background is typically solid white or glassified if layered.
- **Primary Buttons:** Solid Crimson Red (#D90429) background with white text. 12px corner radius. No border.
- **Input Fields:** Minimalist design with no background and a 1px bottom-only border. On focus, the bottom border transitions to Crimson Red.
- **Badges:** Pill-shaped with a very light red background (approx. 10% opacity of primary) and dark red text. Used for status indicators (e.g., "In Transit").
- **Sidebar Nav:** High vertical spacing between items. Text is Dark Charcoal for inactive states, shifting to Crimson Red for the active state accompanied by the vertical indicator line.
- **Data Tables:** Borderless rows with 1px light gray horizontal dividers. Hover states should trigger a subtle #F8F9FA background tint.