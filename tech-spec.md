# Tech Spec — The Benjamin Files

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `react` | `^19.0.0` | UI framework |
| `react-dom` | `^19.0.0` | React DOM renderer |
| `vite` | `^6.2.0` | Build tool |
| `@vitejs/plugin-react` | `^4.4.0` | Vite React plugin |
| `typescript` | `^5.7.0` | Type safety |
| `tailwindcss` | `^4.1.0` | Utility CSS |
| `@tailwindcss/vite` | `^4.1.0` | Tailwind Vite integration |
| `gsap` | `^3.12.0` | Core animation engine, ScrollTrigger |
| `lenis` | `^1.2.0` | Smooth scroll with inertia |
| `@fontsource/syne` | `^5.0.0` | Display font (Hero, Section headers) |
| `@fontsource/space-grotesk` | `^5.0.0` | Body + UI font |
| `@fontsource/space-mono` | `^5.0.0` | Monospace data font |

No shadcn/ui components — the design is fully custom with raw CSS/utility classes.

## Component Inventory

### Layout

| Component | Source | Reuse | Notes |
|-----------|--------|-------|-------|
| `CustomCursor` | Custom | Global | Fixed-position div, follows mouse with lerped position. Scales on hover targets. |
| `SmoothScrollProvider` | Custom | Global | Lenis wrapper — initializes on mount, exposes instance via ref. Connects to GSAP ScrollTrigger. |

### Sections (page-level, used once)

| Component | Source | Notes |
|-----------|--------|-------|
| `HeroSection` | Custom | Full-screen pinned hero. Recording indicator + massive "BENJAMIN" text. ScrollTrigger pins for 200px; text blurs and transitions white→red during pin. |
| `ProfileGallery` | Custom | Horizontal scroll morph gallery. 10 images scroll horizontally via GSAP, collapse into "WHAT HAVE YOU DONE" text. |
| `DataGrid` | Custom | 3-column red-bordered stat grid. Static layout, no complex animation. |
| `AnalysisTimeline` | Custom | Split layout: sticky left headers, scrolling right content. Uses HighlightScrub on key phrases. |
| `VideoEvidenceVault` | Custom | 2-column masonry grid of video cards. Hover interaction with sibling desaturation. |
| `Footer` | Custom | Cascade Grid Entrance typography. |

### Reusable Components

| Component | Source | Used By | Notes |
|-----------|--------|---------|-------|
| `HighlightScrub` | Custom | AnalysisTimeline | Wraps text spans. GSAP ScrollTrigger scrubs color from `#333` → `#fff`. |
| `VideoCard` | Custom | VideoEvidenceVault | Card with red header bar + `<video>` element. Exposes hover state to parent for sibling selection. |

### Hooks

| Hook | Purpose |
|------|---------|
| `useMousePosition` | Tracks mouse coordinates with optional lerp. Used by CustomCursor. |

## Animation Implementation

| Animation | Library | Implementation Approach | Complexity |
|-----------|---------|------------------------|------------|
| **Hero Pin + Color Shift** | GSAP + ScrollTrigger | Pin hero container for 200px. Scrub timeline: blur filter + text color white→red on "BENJAMIN" letters. | 🔒 High |
| **Scroll Gallery Text Morph** | GSAP + ScrollTrigger | Pin wrapper, scrub horizontal xPercent on track. `onUpdate` callback calculates `settleRatio` (0→1), drives per-image scale formula `Math.pow(0.8, index) + (Math.pow(0.8, 9-index) * settleRatio)` and parallax offset. At `settleRatio > 0.99`, swap images for text. | 🔒 High |
| **Highlight Scrub** | GSAP + ScrollTrigger | `ScrollTrigger.batch` or per-element triggers with `scrub: true`. Color tween `#333` → `#fff`. | Low |
| **Video Vault Hover** | GSAP | `mouseenter`/`mouseleave` handlers. Scale hovered card to 1.02, apply grayscale+opacity to siblings via GSAP. | Low |
| **Custom Cursor** | requestAnimationFrame | Lerp position toward mouse coordinates. Scale transform on hover targets via CSS class detection or GSAP. | Medium |
| **Cascade Grid Entrance** | CSS-only | `display: grid; justify-content: start` with incremental `padding-left: Nch`. No JS animation needed — the layout IS the effect. | Low |
| **Smooth Scroll** | Lenis | Global instance, `lerp: 0.15`. Connected to GSAP ScrollTrigger via `lenis.on('scroll', ScrollTrigger.update)`. | Low |

## State & Logic

### Custom Cursor → Hover Target Coordination

The cursor needs to know when it's over an interactive element (video cards, any future links). Two approaches:

1. **CSS-driven**: Interactive elements get a `data-cursor="hover"` attribute. The cursor component queries `document.querySelectorAll('[data-cursor]')` and attaches listeners on mount.
2. **Event delegation**: Single listener on `document` checking `e.target.closest('[data-cursor]')`.

Go with **approach 2** — fewer listeners, simpler. The cursor component handles all logic internally; no external state needed.

### Video Vault Sibling Selection

On `mouseenter` of a `VideoCard`, the parent `VideoEvidenceVault` needs to apply effects to all *other* cards. Approach:

- Parent maintains a `hoveredIndex` state (number | null).
- Pass `isDimmed={hoveredIndex !== null && hoveredIndex !== index}` to each `VideoCard`.
- `VideoCard` uses GSAP to tween its own scale and opacity based on `isDimmed` prop changes.

This avoids complex DOM traversal from the child.

### Scroll Gallery settleRatio

The morph gallery needs a shared `settleRatio` value (0→1) that drives both the image scales and the text/image visibility swap. Since this updates every scroll frame, use a GSAP tween on a proxy object:

```js
const proxy = { value: 0 };
gsap.to(proxy, {
  value: 1,
  scrollTrigger: { ... },
  onUpdate: () => {
    const settleRatio = proxy.value;
    // update all image scales
    // check settleRatio > 0.99 for text swap
  }
});
```

This keeps everything in GSAP's animation loop — no React state involved in the scroll-driven updates.

## Other Key Decisions

### No React Router
Single-page experience. All sections render on one scrollable page. Navigation (if any) uses anchor links that scroll to section IDs.

### Font Loading
Use `@fontsource` packages for self-hosted fonts (no Google Fonts CDN). Import in `main.tsx`:

```tsx
import '@fontsource/syne/400.css';
import '@fontsource/syne/700.css';
import '@fontsource/syne/800.css';
import '@fontsource/space-grotesk/300.css';
import '@fontsource/space-grotesk/400.css';
import '@fontsource/space-grotesk/500.css';
import '@fontsource/space-mono/400.css';
```

### Video Assets
3 looping videos generated as specified in design.md. Place in `public/videos/`. Use `<video>` elements with `autoPlay muted loop playsInline` attributes. No video player library — native HTML5 video is sufficient for autoplaying loops.

### Image Assets
10 gallery images for the morph section. Place in `public/images/gallery/`. These are user-provided photos of Ben. For the build, use placeholder images that the user will swap out later.
