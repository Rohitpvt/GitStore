# GitStore — UI Design System & Component Guidelines

**Document Version:** 1.1.0  
**Phase:** Phase 0.1 — Architecture Audit, Hardening & Final Validation  
**Status:** Approved Design System (Hardened)  
**Design Influence:** Cyberpunk Hardware / Technical Obsidian Aesthetic (Derived from [`Sample Design MD/brew-coffee-shop-DESIGN.md`](./Sample%20Design%20MD/brew-coffee-shop-DESIGN.md) & `https://brew-coffee-shop.aura.build/`)  
**Product Identity:** "The Terminal-Grade Open Software Depot"

---

## 1. Visual Identity & Design Philosophy

GitStore adapts the high-contrast obsidian aesthetic, tech-cut polygons, and monospace technical precision of the reference design system, transforming it into a cutting-edge developer software marketplace.

### Adaptation Principles
- **No Coffee Elements:** All coffee references, organic grain textures, roast notes, and cafeteria idioms are completely eliminated.
- **Developer-Centric Semiotics:** Replaced with compiler flags, Git commit hashes, semver chips, release sizes, platform badges, and terminal-bracketed navigation.
- **Brutalist Meets Glassmorphism:** Deep obsidian background palettes paired with frosted glass panels (`backdrop-filter: blur(16px)`), chamfered 45-degree polygon buttons, and corner crosshair markers evoking hardware schematics and high-precision tooling.

---

## 2. Color Palette & Design Tokens (`variables.css`)

```css
:root {
  /* Surface & Background */
  --bg-primary: #040405;         /* Deep obsidian void */
  --bg-secondary: #0a0a0c;       /* Structural panels & containers */
  --bg-surface: #121216;         /* Elevated cards and interactive panels */
  --bg-glass: rgba(10, 10, 12, 0.75); /* Frosted backdrop blur surface */

  /* Borders & Grids */
  --border-glass: rgba(255, 255, 255, 0.08); /* Standard glass outline */
  --border-focus: #05d9e8;                     /* Cyan focus ring */
  --grid-line: rgba(255, 255, 255, 0.03);      /* 30px ambient blueprint grid */

  /* Neon Accents & Signal Indicators */
  --accent-green: #00ff66;       /* Verified developer, success, active releases */
  --accent-cyan: #05d9e8;        /* Interactive links, primary CTA, platform tags */
  --accent-pink: #ff2a6d;        /* Critical actions, destructive alerts, highlights */
  --accent-amber: #ffb800;       /* Queued status, review pending, warnings */

  /* Text & Typography */
  --text-primary: #f0f0f0;       /* High-contrast crisp body & headings */
  --text-muted: #888888;         /* Secondary descriptions and metadata */
  --text-dim: #44444c;           /* Disabled states and subtle dividers */

  /* Typography Scales */
  --font-display: "Syne", "Inter", sans-serif;
  --font-body: "Space Grotesk", "Inter", sans-serif;
  --font-mono: "JetBrains Mono", "Fira Code", monospace;

  /* Spacing Scale */
  --space-2xs: 4px;
  --space-xs: 8px;
  --space-sm: 12px;
  --space-md: 24px;
  --space-lg: 48px;
  --space-xl: 80px;
  --container-max: 1280px;

  /* Tech Polygons & Corner Cuts */
  --tech-cut-sm: polygon(0 0, calc(100% - 8px) 0, 100% 8px, 100% 100%, 8px 100%, 0 calc(100% - 8px));
  --tech-cut-lg: polygon(0 0, calc(100% - 16px) 0, 100% 16px, 100% 100%, 16px 100%, 0 calc(100% - 16px));
}
```

---

## 3. Typography & Hierarchy

1. **Hero Display Heading:**
   - Font: `Syne`, 800 Weight, uppercase, tracking `-0.03em`.
   - Size: `clamp(3rem, 7vw, 6.5rem)`.
   - Used for the primary homepage title: "DISCOVER OPEN SOFTWARE. DIRECT FROM SOURCE."
2. **Section Headings:**
   - Font: `Syne`, 700 Weight, uppercase.
   - Size: `clamp(1.75rem, 4vw, 2.75rem)`.
   - Preceded by a monospace section indicator: `// SECTION: FEATURED_RELEASES`.
3. **Card & Product Titles:**
   - Font: `Space Grotesk`, 600 Weight.
   - Size: `1.25rem` (20px), line-height `1.3`.
4. **Technical & Metadata Labels:**
   - Font: `JetBrains Mono`, 400 Weight.
   - Size: `0.75rem` (12px), uppercase, tracking `0.05em`.
   - Example: `TAG: v2.4.1 | EXT: .EXE | SIZE: 42.8 MB | LIC: MIT`.

---

## 4. Reusable Core UI Components

### 4.1 Fixed Obsidian Navigation (`navigation.css`)
- **Visuals:** Fixed header with blurred obsidian background (`backdrop-filter: blur(20px)`), bottom glass border (`rgba(255,255,255,0.08)`).
- **Branding:** `GitStore // [STATION_01]` in monospace green/cyan.
- **Bracketed Links:** Interactive nav links rendered with character brackets that highlight on hover:
  - `[ EXPLORE ]`
  - `[ CATEGORIES ]`
  - `[ SUBMIT APP ]`
  - `[ DASHBOARD ]`
- **Right Action:** Tech-cut CTA button `[ CONNECT GITHUB ]` with neon cyan border and glowing hover state.

### 4.2 Product Showcase Card (`cards.css`)
- **Structure:**
  - Tech-cut polygon box or rounded glass panel with corner crosshair `+` markers in all four corners.
  - Header: Application icon, product name, and verified developer pill `[VERIFIED_DEV]`.
  - Body: 2-line short description (`--text-muted`).
  - Tags: Platform pills (Windows, macOS, Linux, Android) and version tag (`v1.4.0`).
  - Footer: GitHub stars counter, asset size (`48.2 MB`), and direct download button.

```html
<div class="product-card tech-container">
  <span class="crosshair top-left">+</span>
  <span class="crosshair top-right">+</span>
  <span class="crosshair bottom-left">+</span>
  <span class="crosshair bottom-right">+</span>
  
  <div class="card-header">
    <img src="icon.png" alt="App Icon" class="app-icon" />
    <div class="title-meta">
      <h3 class="app-title">SuperTerminal</h3>
      <span class="badge verified">[ VERIFIED_DEV ]</span>
    </div>
  </div>

  <p class="app-desc">High-speed GPU-accelerated terminal emulator for Windows and Linux.</p>

  <div class="card-pills">
    <span class="pill">WINDOWS</span>
    <span class="pill">LINUX</span>
    <span class="pill tag">v3.2.1</span>
  </div>

  <div class="card-footer">
    <span class="stars-count">★ 2,410</span>
    <span class="asset-size">48.2 MB</span>
    <a href="..." class="btn btn-primary-chamfer">DOWNLOAD .EXE</a>
  </div>
</div>
```

### 4.3 Chamfered Tech Buttons (`buttons.css`)
- **Shape:** 45-degree corner chamfer via `clip-path: var(--tech-cut-sm)`.
- **Variants:**
  - `.btn-primary`: Neon Cyan background, obsidian text, bold tracking.
  - `.btn-accent`: Neon Pink outline with pink glow on hover.
  - `.btn-terminal`: Monospace bracketed action with green blinking indicator.

### 4.4 Status Badges & Lifecycle Chips (`badges.css`)
Monospace font, uppercase, strict semantic color coding:
- `[ VERIFIED_DEV ]`: `--accent-green` background with dark text.
- `[ QUEUED ]`: `--accent-amber` text with glass border.
- `[ PROCESSING ]`: `--accent-cyan` animated pulse indicator.
- `[ NEEDS_REVIEW ]`: `--accent-amber` pulse border.
- `[ APPROVED ]`: `--accent-green` text with glass border.
- `[ PUBLISHED ]`: `--accent-green` solid fill.
- `[ REJECTED ]`: `--accent-pink` border with warning icon.
- `[ FAILED ]`: `--accent-pink` solid text.

### 4.5 Legal Non-Execution Disclaimer Banner (`product-detail.css`)
Every product detail view (`product.html`) features a prominent schematic disclaimer panel:
```html
<div class="disclaimer-panel tech-container">
  <span class="crosshair top-left">+</span>
  <span class="crosshair top-right">+</span>
  <p class="disclaimer-text">
    <span class="highlight">// DISCLAIMER:</span> Downloads link directly to official upstream GitHub Releases assets. GitStore verifies file headers and public source existence, but does NOT execute, sandbox, or audit compiled binary code. Verify source code and run local anti-malware scanners prior to execution.
  </p>
</div>
```

### 4.6 Decoupled Admin Moderation Component (`admin.css`)
In the Admin Dashboard (`admin.html`), actions do not execute client-side Admin SDK calls. Instead, the UI renders an asynchronous command state:
- **Button State:** `[ ENQUEUE APPROVAL ]`
- **In-Flight Indicator:** `[ STATUS: ACTION QUEUED IN ACTIONS WORKER ]` with a direct link to the GitHub Actions workflow for immediate dispatch.
- **Live Response:** Reactively changes to `[ STATUS: PUBLISHED LIVE ]` via Firestore `onSnapshot`.

---

## 5. Responsive Layout & Breakpoints

- **Desktop (>= 1024px):** 3-column or 4-column product grid with persistent sticky left filter drawer.
- **Tablet (768px – 1023px):** 2-column product grid; collapsible filter panel accessible via a toggle button.
- **Mobile (< 768px):** Single-column stacked cards, full-width touch-friendly download buttons, slide-in navigation drawer.

---

## 6. Accessibility & Motion Standards

1. **High Contrast Compliance:** All text styles meet WCAG 2.1 AA standards (minimum 4.5:1 contrast ratio against the obsidian background).
2. **Keyboard Navigation:** Explicit cyan focus rings (`outline: 2px solid var(--accent-cyan); outline-offset: 3px`) for all focusable buttons, inputs, and links.
3. **Reduced Motion:** All hover translations, scans, and scramble text effects respect `prefers-reduced-motion`:
   ```css
   @media (prefers-reduced-motion: reduce) {
     * {
       animation-duration: 0.01ms !important;
       transition-duration: 0.01ms !important;
     }
   }
   ```
