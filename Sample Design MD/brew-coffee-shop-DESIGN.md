---
version: 2.0.25
name: BREW_SYSTEM
description: A futuristic, high-contrast visual system for specialty coffee establishments.
colors:
  bg-primary: "#040405"
  bg-secondary: "#0a0a0c"
  accent-green: "#00ff66"
  accent-pink: "#ff2a6d"
  accent-cyan: "#05d9e8"
  text-primary: "#f0f0f0"
  text-muted: "#888888"
  glass-border: "rgba(255,255,255,0.08)"
typography:
  display-hero:
    fontFamily: "Syne"
    fontSize: "12rem"
    fontWeight: 800
    lineHeight: "1"
  heading-section:
    fontFamily: "Syne"
    fontSize: "4rem"
    fontWeight: 700
    lineHeight: "1.1"
  body-standard:
    fontFamily: "Space Grotesk"
    fontSize: "1rem"
    fontWeight: 400
    lineHeight: "1.6"
  technical-label:
    fontFamily: "Noto Sans JP"
    fontSize: "0.75rem"
    fontWeight: 400
    lineHeight: "1.4"
spacing:
  xs: "4px"
  sm: "12px"
  md: "24px"
  lg: "48px"
  xl: "80px"
  container-max: "1200px"
rounded:
  none: "0px"
  sm: "6px"
  md: "10px"
  lg: "12px"
  full: "999px"
components:
  navigation:
    style: "Fixed glass header with character-bracketed links and clipped CTA."
  hero:
    style: "Centered vertical stack with character-split reveal animations."
  menu-item:
    style: "Horizontal row with leading image, dotted price leader, and technical order button."
  capability-card:
    style: "Grid based container with left-border hover indicator and scrambled text labels."
  work-card:
    style: "Grayscale-to-color image transition with gradient overlay and bottom-aligned labels."
  cta-panel:
    style: "Glassmorphic box with corner crosshairs and 'SYSTEM_READY' label indicator."
motion:
  lerp: 0.1
  hover-ease: "expo.out"
  scramble-speed: "30ms"
---

## Overview
BREW. is a visual language that merges third-wave coffee culture with cyberpunk hardware aesthetics. It prioritizes technical precision, high contrast, and "terminal-style" interactions.

## Colors
The palette is rooted in deep obsidian blacks. Primary accents are high-vibrancy neons (Green, Pink, Cyan) used to signify interactive states, system status, and product categorization.

## Typography
Features a trio of typefaces: Syne for brutalist display impact, Space Grotesk for technical body readability, and Noto Sans JP for cultural flavoring and vertical labels.

## Spacing
Utilizes a 30px grid background as a spatial reference. Padding and margins are aggressive, often using clamp() to ensure dramatic scaling on larger displays.

## Layout
Layouts use a mix of CSS Grid and Flexbox with intentional vertical breaks. Grid lines are often visible or implied through 'glass' borders and crosshair markers.

## Elevation & Depth
Depth is achieved through glassmorphism (backdrop-blur: 16px-20px) rather than soft shadows. Elevation is marked by border-color intensity and subtle z-axis shifts during hover.

## Shapes
Heavy use of 'tech-cut' polygons. Clipping paths are used for buttons and cards to create 45-degree chamfered corners instead of standard rounded edges.

## Components
- **Nav**: Fixed-top, transitions from transparent to blurred obsidian on scroll.
- **Menu Item**: Includes a leading image with grayscale filters that activate on hover.
- **Order Button**: High-contrast rose/pink tinted glass with ultra-tight tracking.
- **Crosshairs**: Absolute positioned '+' markers at container corners to evoke hardware schematics.

## Motion
Motion is characterized by 'mechanical' reveals. Text should scramble or translate vertically (Y-axis) with exponential easing. Inertial scrolling (Lenis) is core to the system feel.

## Do's and Don'ts
- Do use Noto Sans JP for purely aesthetic, low-opacity vertical labels.
- Do use 'Golden Grains' particle effects for organic texture.
- Don't use standard border-radius larger than 12px; prefer polygon clips.
- Don't use warm earth tones; stick to cold obsidian and neon highlights.

## Accessibility
Focus states must be high-contrast (Accent Cyan). Motion must be disabled via prefers-reduced-motion media queries. Aria-labels are required for all icon-only terminal buttons.