# Tech Spec — Kondapalli Sai Hiteesh Portfolio

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| react | ^19 | UI framework |
| react-dom | ^19 | DOM renderer |
| vite | ^6 | Build tool |
| @vitejs/plugin-react | ^4 | Vite React plugin |
| typescript | ^5 | Type safety |
| tailwindcss | ^4 | Utility CSS |
| @tailwindcss/vite | ^4 | Tailwind Vite integration |
| gsap | ^3.12 | Animation engine, ScrollTrigger, SplitText |
| lenis | ^1.2 | Smooth scroll |
| lucide-react | ^0.460 | Icons |

No shadcn/ui components — the design is fully bespoke editorial.

## Component Inventory

### Layout (shared)

| Component | Source | Notes |
|-----------|--------|-------|
| Navigation | Custom | Fixed top bar. Transparent → blurred navy on scroll (80px threshold). Mobile: hamburger → fullscreen overlay. |
| Footer | Custom | Saffron background, 4-column grid (nav, connect, location, certifications). Mobile: single column stack. |
| GrainOverlay | Custom | Fixed fullscreen canvas, `pointer-events: none`, `z-index: 9999`. Random pixel grain redrawn every 100ms at 0.06 opacity. |

### Sections

| Component | Source | Notes |
|-----------|--------|-------|
| HeroSection | Custom | Full viewport. Hosts WordShuffle + background blob + stats row + scroll indicator. |
| AboutSection | Custom | Two-column: left text, right expertise cards (4 cards with lucide icons). |
| ExperienceSection | Custom | Cream background. Central timeline line with alternating left/right cards. Single company entry (Applaunch). |
| ProjectsSection | Custom | 2-column grid of 6 project cards with generated images/overlays. Cards 5 & 6 use gradient backgrounds. |
| SkillsSection | Custom | 2-column grid of 5 skill category cards (tag lists) + horizontal certification row. |
| ContactSection | Custom | Hosts AmbientTrailCanvas as background. CTAs, contact grid, resume download link. |

### Reusable Components

| Component | Source | Used By |
|-----------|--------|---------|
| SectionLabel | Custom | All sections — "ABOUT", "EXPERIENCE", etc. Caption text + thin accent line. |
| SectionHeadline | Custom | All sections — 92px serif headline with optional transparent stroke + saffron period. |
| WordShuffle | Custom | HeroSection — letter-by-letter shuffle animation cycling through 4 roles. |
| AmbientTrailCanvas | Custom | ContactSection — Canvas 2D particle system with mouse repel, proximity lines, dazzle effect. |

## Animation Implementation

| Animation | Library | Implementation Approach | Complexity |
|-----------|---------|------------------------|------------|
| Hero word shuffle (letter stagger in/out) | GSAP | `useEffect` interval (3s) + GSAP timeline per cycle. Split word into letter spans. Outgoing: opacity→0, y→random. Incoming: opacity→1, y→0, staggered from center (15ms × distance from center). | **High** 🔒 |
| Hero entrance sequence | GSAP timeline | Single timeline: blob scale→1, label fade-in, letter stagger (SplitText-like manual span mapping), subtitle fade, stats stagger, scroll indicator. | Medium |
| Scroll-triggered section reveals | GSAP ScrollTrigger | Reusable helper: headlines (y:40→0, opacity, 0.8s), body (y:30→0, 0.7s, 0.15s delay), cards/images (y:50→0, 0.9s, 0.3s delay). Stagger 0.12s. `start: "top 78%"`, `toggleActions: "play none none none"`. | Medium |
| About cards stagger from right | GSAP ScrollTrigger | Cards: x:40→0, opacity, stagger 0.15s. `start: "top 75%"`. | Low |
| Experience timeline cards (alternating sides) | GSAP ScrollTrigger | Left cards: x:-50→0, right cards: x:50→0, stagger 0.2s. `start: "top 80%"`. | Medium |
| Project cards stagger | GSAP ScrollTrigger | y:60→0, opacity, stagger 0.15s. `start: "top 80%"`. | Low |
| Skills cards stagger | GSAP ScrollTrigger | y:40→0, opacity, stagger 0.12s. `start: "top 80%"`. | Low |
| Contact entrance | GSAP ScrollTrigger | Headline y:40→0, subtext (+0.15s), CTAs (+0.3s, stagger 0.1s), grid (+0.5s). | Low |
| Nav background transition | CSS | `transition: background 0.4s, backdrop-filter 0.4s`. Scroll listener toggles class at 80px. | Low |
| Nav link underline hover | CSS | `scaleX(0)→scaleX(1)`, `transform-origin: left`, `transition: 0.3s cubic-bezier(0.16,1,0.3,1)`. | Low |
| Card hover lift + shadow | CSS | `translateY(-6px)`, `box-shadow`, `transition: 0.4s cubic-bezier(0.16,1,0.3,1)`. Inner image `scale(1.05)`, `transition: 0.6s`. | Low |
| Button hover glow | CSS | `scale(1.04)`, saffron glow shadow, `transition: 0.25s`. | Low |
| Scroll indicator bounce | CSS @keyframes | `translateY` bounce, 2s infinite ease. | Low |
| Grain texture | Canvas 2D (custom) | Fixed canvas, `setInterval` 100ms random pixel fill, `mix-blend-mode: overlay`, opacity 0.06. | Low |
| Cursor ambient trail | Canvas 2D (custom) | **🔒 AmbientTrailCanvas**: 80 Dot instances with Brownian drift, elastic wraparound, velocity-based mouse repel (150px radius, 2.5 force), proximity lines (120px, 0.08 alpha, mouse-idle only), glow halos on hovered dots, dazzle on link hover (random dot expands to 5-9px, saffron glow, 800ms). `IntersectionObserver` pauses rAF loop when offscreen. | **High** 🔒 |
| Mobile hamburger menu | CSS + React state | Full-screen navy overlay, links stacked vertically. Open/close state toggled by hamburger click. | Low |
| Smooth scrolling | Lenis | `lerp: 0.08`, `duration: 1.0`. `lenis.on('scroll', ScrollTrigger.update)` on every frame. Nav clicks: `lenis.scrollTo(target, { offset: -80 })`. | Low |

## State & Logic

### WordShuffle State Machine

React state (not GSAP): `wordIndex`, `isAnimatingOut`, `isAnimatingIn`. `setInterval` every 3000ms drives the cycle:

1. Set `isAnimatingOut = true`
2. GSAP animates current letters out (400ms)
3. After 400ms: increment `wordIndex`, set `isAnimatingIn = true`
4. GSAP animates new letters in (500ms)
5. After 600ms total: clear flags, word is settled

Requires manual letter span mapping (no SplitText plugin — manual React span rendering of each character). Words are split into arrays of letter objects with unique keys to enable GSAP targeting.

### AmbientTrailCanvas Architecture

All animation state lives inside the Canvas component (no React re-renders):
- `Dot[]` array instantiated once on mount
- `mouse` position tracked via `mousemove` listener on canvas
- `mouseIdle` boolean: `mousemove` sets false, `setTimeout` (100ms) sets true
- `requestAnimationFrame` loop: `update()` all dots → `drawLines()` (if idle) → `drawDots()`
- Dazzle: exposed as a callback ref (`triggerDazzle()`) that contact links call on `mouseenter`
- `IntersectionObserver` on canvas container: pauses/resumes rAF loop
- Resize handler: recalculates canvas dimensions, repositions dots proportionally

### Reduced Motion

`@media (prefers-reduced-motion: reduce)` disables: word shuffle (static display), canvas animation (static dot render), scroll-triggered reveals (GSAP timelines killed, content shown immediately with CSS `opacity: 1; transform: none`). Lenis `smoothWheel: false`.

## Other Key Decisions

- **No routing** — single page, anchor links only. Navigation uses `lenis.scrollTo()`.
- **No dark mode toggle** — the design is inherently dark-mode (navy backgrounds with cream text).
- **Audio element**: Hidden `<audio>` tag for ambient background music, toggled by a fixed circular button (bottom-right). Auto-play on first user click (browser autoplay policy).
- **Image assets**: 4 generated images (project-mri, project-listy, project-polingo, project-malusaki). Used as CSS `background-image` on project cards with `object-fit: cover`.
