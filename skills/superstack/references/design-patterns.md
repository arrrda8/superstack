# Design Patterns Reference — Superstack

## Table of Contents
- [1. Lenis Smooth Scroll Setup (Next.js)](#1-lenis-smooth-scroll-setup-nextjs)
  - [Provider Component](#provider-component)
  - [Layout Integration](#layout-integration)
  - [Parallax Component](#parallax-component)
- [2. OKLCH Color System](#2-oklch-color-system)
- [3. Noise/Grain Texture](#3-noisegrain-texture)
  - [Inline SVG Noise Filter](#inline-svg-noise-filter)
  - [CSS Overlay Technique](#css-overlay-technique)
- [4. AI Anti-Pattern Fixes](#4-ai-anti-pattern-fixes)
  - [Staggered Entrance Animations](#staggered-entrance-animations)
  - [Text Reveal with clip-path](#text-reveal-with-clip-path)
  - [Magnetic Hover (CSS-only approximation)](#magnetic-hover-css-only-approximation)
- [5. Layout Patterns](#5-layout-patterns)
  - [Asymmetric Split (40/60)](#asymmetric-split-4060)
  - [Bento Grid](#bento-grid)
  - [Overlapping Grid](#overlapping-grid)
  - [Horizontal Scroll Section](#horizontal-scroll-section)
- [6. Typography Techniques](#6-typography-techniques)
  - [Font Stack Setup](#font-stack-setup)
  - [Mixed-Weight Headlines](#mixed-weight-headlines)
  - [Monospace Accents](#monospace-accents)
- [7. Framer Motion Recipes](#7-framer-motion-recipes)
  - [Staggered Reveal (whileInView)](#staggered-reveal-whileinview)
  - [Magnetic Button](#magnetic-button)
  - [Smooth Page Transitions](#smooth-page-transitions)
  - [Card Hover with 3D Perspective](#card-hover-with-3d-perspective)
  - [Scroll-Linked Animation (useScroll + useTransform)](#scroll-linked-animation-usescroll-usetransform)
  - [Spring Physics (useSpring)](#spring-physics-usespring)
- [8. GSAP Scroll Patterns](#8-gsap-scroll-patterns)
  - [ScrollTrigger Timeline with Pinning and Snapping](#scrolltrigger-timeline-with-pinning-and-snapping)
  - [SplitText Character Animation](#splittext-character-animation)
  - [Parallax Layers (different speeds)](#parallax-layers-different-speeds)
  - [Number Counter Animation](#number-counter-animation)
  - [Text Reveal Line-by-Line](#text-reveal-line-by-line)
- [9. Modern CSS Techniques](#9-modern-css-techniques)
  - [Container Queries (95% support)](#container-queries-95-support)
  - [:has() Selector (93%)](#has-selector-93)
  - [Scroll-Driven Animations (85%)](#scroll-driven-animations-85)
  - [View Transitions API (83%)](#view-transitions-api-83)
  - [Subgrid (93%)](#subgrid-93)
  - [@property for Animated Gradients](#property-for-animated-gradients)
  - [CSS Masonry (2026)](#css-masonry-2026)
  - [@supports Progressive Enhancement](#supports-progressive-enhancement)
- [10. Component Recipes](#10-component-recipes)
  - [Glassmorphism Card](#glassmorphism-card)
  - [Gradient Border (::before technique)](#gradient-border-before-technique)
  - [Spotlight Effect (cursor-following glow)](#spotlight-effect-cursor-following-glow)
  - [Custom Scrollbar + Selection Styling](#custom-scrollbar-selection-styling)
- [11. Hero Section Variants](#11-hero-section-variants)
  - [1. Product-Led Hero](#1-product-led-hero)
  - [2. Text-First Hero](#2-text-first-hero)
  - [3. Split Screen Hero](#3-split-screen-hero)
  - [4. Video Background Hero](#4-video-background-hero)
  - [5. Interactive Hero (cursor-responsive)](#5-interactive-hero-cursor-responsive)
- [12. Navigation Patterns](#12-navigation-patterns)
  - [Full-Screen Overlay (clip-path circle reveal)](#full-screen-overlay-clip-path-circle-reveal)
  - [Floating Pill Nav (blurred background, centered)](#floating-pill-nav-blurred-background-centered)
  - [Morphing Header (shrinks on scroll)](#morphing-header-shrinks-on-scroll)
  - [Bottom Sheet Navigation (mobile)](#bottom-sheet-navigation-mobile)
- [13. Section Designs](#13-section-designs)
  - [Pricing Table with Highlighted Plan](#pricing-table-with-highlighted-plan)
  - [FAQ Accordion with Animations](#faq-accordion-with-animations)
  - [Stats Section with Animated Counters](#stats-section-with-animated-counters)
  - [Footer with Newsletter + Social](#footer-with-newsletter-social)
- [14. Magic UI / Aceternity UI Pattern Catalog](#14-magic-ui-aceternity-ui-pattern-catalog)
  - [Recommended Components by Category](#recommended-components-by-category)
  - [Usage Guidelines](#usage-guidelines)
- [15. Premium CSS Checklist](#15-premium-css-checklist)
- [Quick Reference: Custom Bezier Curves](#quick-reference-custom-bezier-curves)

Production-ready patterns for premium web projects. Every example is copy-pasteable.

---

## 1. Lenis Smooth Scroll Setup (Next.js)

```bash
bun add lenis gsap @gsap/react
```

### Provider Component

```tsx
// components/smooth-scroll.tsx
"use client";

import { ReactLenis, useLenis } from "lenis/react";
import { useEffect } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";

gsap.registerPlugin(ScrollTrigger);

export function SmoothScroll({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    // Critical: disable GSAP lag smoothing for Lenis compatibility
    gsap.ticker.lagSmoothing(0);
  }, []);

  return (
    <ReactLenis
      root
      options={{
        lerp: 0.1,
        duration: 1.2,
        smoothWheel: true,
        wheelMultiplier: 1,
        touchMultiplier: 2,
      }}
    >
      {children}
    </ReactLenis>
  );
}

// Hook to sync Lenis with ScrollTrigger (use in layout or page)
export function useLenisScrollTrigger() {
  useLenis((lenis) => {
    // Sync Lenis scroll position to ScrollTrigger
    ScrollTrigger.update();
  });

  useEffect(() => {
    const update = (time: number) => {};
    gsap.ticker.add(update);
    return () => gsap.ticker.remove(update);
  }, []);
}
```

### Layout Integration

```tsx
// app/layout.tsx
import { SmoothScroll } from "@/components/smooth-scroll";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="de">
      <body>
        <SmoothScroll>{children}</SmoothScroll>
      </body>
    </html>
  );
}
```

### Parallax Component

```tsx
"use client";

import { useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

gsap.registerPlugin(ScrollTrigger);

export function Parallax({
  children,
  speed = 0.5,
  className,
}: {
  children: React.ReactNode;
  speed?: number;
  className?: string;
}) {
  const ref = useRef<HTMLDivElement>(null);

  useGSAP(() => {
    if (!ref.current) return;
    gsap.to(ref.current, {
      yPercent: speed * 100,
      ease: "none",
      scrollTrigger: {
        trigger: ref.current,
        start: "top bottom",
        end: "bottom top",
        scrub: true,
      },
    });
  }, [speed]);

  return (
    <div ref={ref} className={className}>
      {children}
    </div>
  );
}
```

---

## 2. OKLCH Color System

```css
/* globals.css — Full OKLCH Design System */
:root {
  /* Brand hue — change this single value to re-theme everything */
  --hue: 264;
  --chroma: 0.15;

  /* Brand palette — shift lightness only */
  --brand-50:  oklch(0.97 0.03 var(--hue));
  --brand-100: oklch(0.93 0.06 var(--hue));
  --brand-200: oklch(0.87 0.09 var(--hue));
  --brand-300: oklch(0.78 0.12 var(--hue));
  --brand-400: oklch(0.68 var(--chroma) var(--hue));
  --brand-500: oklch(0.58 var(--chroma) var(--hue));
  --brand-600: oklch(0.48 var(--chroma) var(--hue));
  --brand-700: oklch(0.38 0.12 var(--hue));
  --brand-800: oklch(0.28 0.09 var(--hue));
  --brand-900: oklch(0.18 0.06 var(--hue));

  /* Neutrals with brand hue tint (low chroma) */
  --neutral-50:  oklch(0.98 0.005 var(--hue));
  --neutral-100: oklch(0.94 0.008 var(--hue));
  --neutral-200: oklch(0.88 0.01 var(--hue));
  --neutral-300: oklch(0.78 0.01 var(--hue));
  --neutral-400: oklch(0.63 0.01 var(--hue));
  --neutral-500: oklch(0.50 0.01 var(--hue));
  --neutral-600: oklch(0.40 0.01 var(--hue));
  --neutral-700: oklch(0.30 0.01 var(--hue));
  --neutral-800: oklch(0.20 0.008 var(--hue));
  --neutral-900: oklch(0.13 0.005 var(--hue));

  /* Semantic tokens */
  --bg: var(--neutral-50);
  --bg-subtle: var(--neutral-100);
  --text: var(--neutral-900);
  --text-muted: var(--neutral-500);
  --border: var(--neutral-200);
  --accent: var(--brand-500);
  --accent-hover: color-mix(in oklch, var(--brand-500), white 15%);
  --accent-active: color-mix(in oklch, var(--brand-500), black 10%);

  /* Surface layers */
  --surface-1: var(--neutral-50);
  --surface-2: var(--neutral-100);
  --surface-3: var(--neutral-200);
}

/* Dark mode — flip lightness */
.dark, [data-theme="dark"] {
  --bg: var(--neutral-900);
  --bg-subtle: var(--neutral-800);
  --text: var(--neutral-50);
  --text-muted: var(--neutral-400);
  --border: var(--neutral-700);
  --accent: var(--brand-400);
  --accent-hover: color-mix(in oklch, var(--brand-400), white 20%);
  --accent-active: color-mix(in oklch, var(--brand-400), black 15%);
  --surface-1: var(--neutral-900);
  --surface-2: var(--neutral-800);
  --surface-3: var(--neutral-700);
}

/* Fallback for browsers without oklch (< 2%) */
@supports not (color: oklch(0.5 0.2 264)) {
  :root {
    --brand-500: hsl(264 60% 50%);
    --neutral-900: hsl(264 5% 10%);
  }
}
```

---

## 3. Noise/Grain Texture

### Inline SVG Noise Filter

```html
<!-- Place once in layout, hidden -->
<svg class="hidden" aria-hidden="true">
  <filter id="noise">
    <feTurbulence type="fractalNoise" baseFrequency="0.65" numOctaves="3" stitchTiles="stitch" />
    <feColorMatrix type="saturate" values="0" />
  </filter>
</svg>
```

### CSS Overlay Technique

```css
/* Noise overlay — use on hero, cards, feature sections */
.noise-overlay {
  position: relative;
  isolation: isolate;
}

.noise-overlay::after {
  content: "";
  position: absolute;
  inset: 0;
  filter: url(#noise);
  opacity: 0.04;
  pointer-events: none;
  z-index: 1;
  mix-blend-mode: overlay;
}

/* Glassmorphism with texture */
.glass-card {
  background: color-mix(in oklch, var(--surface-1), transparent 20%);
  backdrop-filter: blur(16px) saturate(1.5);
  border: 1px solid color-mix(in oklch, var(--border), transparent 50%);
  border-radius: 1rem;
  position: relative;
  overflow: hidden;
}

.glass-card::after {
  content: "";
  position: absolute;
  inset: 0;
  filter: url(#noise);
  opacity: 0.03;
  pointer-events: none;
  mix-blend-mode: overlay;
}
```

---

## 4. AI Anti-Pattern Fixes

| Anti-Pattern | Premium Alternative |
|---|---|
| Linear gradients (blue-to-purple) | Solid color with noise texture overlay |
| Uniform card grid | Bento grid with varied sizes |
| Generic rounded corners (rounded-xl) | Intentional radius scale (4px, 8px, 16px, 24px) |
| Drop shadow everywhere | Subtle border + 1 elevated element |
| Centered everything | Asymmetric splits (40/60) |
| Stock illustration style | Photography with duotone filter |
| Rainbow gradients | Monochrome + single accent |
| Icon in circle pattern | Larger icons, no container circle |

### Staggered Entrance Animations

```css
.stagger-list > * {
  opacity: 0;
  transform: translateY(20px);
  animation: stagger-in 0.6s cubic-bezier(0.16, 1, 0.3, 1) forwards;
}

.stagger-list > *:nth-child(1) { animation-delay: 0ms; }
.stagger-list > *:nth-child(2) { animation-delay: 80ms; }
.stagger-list > *:nth-child(3) { animation-delay: 160ms; }
.stagger-list > *:nth-child(4) { animation-delay: 240ms; }
.stagger-list > *:nth-child(5) { animation-delay: 320ms; }
.stagger-list > *:nth-child(6) { animation-delay: 400ms; }

@keyframes stagger-in {
  to { opacity: 1; transform: translateY(0); }
}
```

### Text Reveal with clip-path

```css
.text-reveal {
  clip-path: inset(0 100% 0 0);
  animation: reveal 0.8s cubic-bezier(0.77, 0, 0.18, 1) forwards;
}

@keyframes reveal {
  to { clip-path: inset(0 0% 0 0); }
}
```

### Magnetic Hover (CSS-only approximation)

```css
.magnetic-btn {
  transition: transform 0.3s cubic-bezier(0.33, 1, 0.68, 1);
}
.magnetic-btn:hover {
  transform: scale(1.05);
}
.magnetic-btn:active {
  transform: scale(0.97);
  transition-duration: 0.1s;
}
```

---

## 5. Layout Patterns

### Asymmetric Split (40/60)

```tsx
<section className="grid grid-cols-1 lg:grid-cols-[2fr_3fr] gap-12 lg:gap-20 items-center px-6 py-24">
  <div className="space-y-6">
    <h2 className="text-4xl font-bold tracking-tight">Headline here</h2>
    <p className="text-lg text-[var(--text-muted)] leading-relaxed">Description text.</p>
  </div>
  <div className="relative aspect-[4/3] rounded-2xl overflow-hidden">
    <Image src="/feature.webp" alt="" fill className="object-cover" />
  </div>
</section>
```

### Bento Grid

```tsx
<div className="grid grid-cols-2 md:grid-cols-4 gap-4 p-6">
  <div className="col-span-2 row-span-2 rounded-2xl bg-[var(--surface-2)] p-8">Large card</div>
  <div className="rounded-2xl bg-[var(--surface-2)] p-6">Small</div>
  <div className="rounded-2xl bg-[var(--surface-2)] p-6">Small</div>
  <div className="col-span-2 rounded-2xl bg-[var(--surface-2)] p-6">Wide card</div>
</div>
```

### Overlapping Grid

```css
.overlap-grid {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
}
.overlap-grid .media {
  grid-column: 1 / 8;
  grid-row: 1;
}
.overlap-grid .content {
  grid-column: 6 / 12;
  grid-row: 1;
  z-index: 1;
  align-self: center;
  padding: 3rem;
  background: var(--surface-1);
  border-radius: 1rem;
  margin-block: 4rem;
}
```

### Horizontal Scroll Section

```tsx
<section className="overflow-hidden py-24">
  <div className="flex gap-6 overflow-x-auto snap-x snap-mandatory px-6 scrollbar-hide
                  [&>*]:snap-center [&>*]:shrink-0 [&>*]:w-[80vw] [&>*]:md:w-[40vw]">
    {items.map((item) => (
      <Card key={item.id} {...item} />
    ))}
  </div>
</section>
```

---

## 6. Typography Techniques

### Font Stack Setup

```css
/* globals.css */
:root {
  --font-display: "Clash Display", "Cabinet Grotesk", sans-serif;
  --font-body: "Satoshi", "Plus Jakarta Sans", "Inter", sans-serif;
  --font-mono: "JetBrains Mono", "Fira Code", monospace;
}

/* Fluid typography scale */
:root {
  --text-xs:   clamp(0.75rem, 0.7rem + 0.15vw, 0.8rem);
  --text-sm:   clamp(0.825rem, 0.78rem + 0.2vw, 0.9rem);
  --text-base: clamp(0.9rem, 0.85rem + 0.25vw, 1.05rem);
  --text-lg:   clamp(1.1rem, 1rem + 0.4vw, 1.3rem);
  --text-xl:   clamp(1.25rem, 1.1rem + 0.6vw, 1.6rem);
  --text-2xl:  clamp(1.5rem, 1.2rem + 1vw, 2rem);
  --text-3xl:  clamp(1.875rem, 1.4rem + 1.5vw, 2.5rem);
  --text-4xl:  clamp(2.25rem, 1.6rem + 2.2vw, 3.5rem);
  --text-5xl:  clamp(3rem, 2rem + 3.5vw, 5rem);
}
```

### Mixed-Weight Headlines

```tsx
<h1 className="text-[var(--text-5xl)] font-[var(--font-display)] tracking-[-0.03em] leading-[0.95]">
  <span className="font-light text-[var(--text-muted)]">Build </span>
  <span className="font-bold">faster.</span>
  <br />
  <span className="font-light text-[var(--text-muted)]">Ship </span>
  <span className="font-bold">smarter.</span>
</h1>
```

### Monospace Accents

```tsx
<span className="font-mono text-xs tracking-widest uppercase text-[var(--accent)]">
  // 01 — Features
</span>
```

---

## 7. Framer Motion Recipes

### Staggered Reveal (whileInView)

```tsx
"use client";

import { motion } from "framer-motion";

const container = {
  hidden: {},
  show: { transition: { staggerChildren: 0.08 } },
};

const item = {
  hidden: { opacity: 0, y: 24 },
  show: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.5, ease: [0.16, 1, 0.3, 1] },
  },
};

export function StaggerGrid({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      variants={container}
      initial="hidden"
      whileInView="show"
      viewport={{ once: true, margin: "-100px" }}
      className="grid grid-cols-1 md:grid-cols-3 gap-6"
    >
      {Array.isArray(children)
        ? children.map((child, i) => (
            <motion.div key={i} variants={item}>{child}</motion.div>
          ))
        : children}
    </motion.div>
  );
}
```

### Magnetic Button

```tsx
"use client";

import { motion, useMotionValue, useSpring } from "framer-motion";
import { useRef } from "react";

export function MagneticButton({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLButtonElement>(null);
  const x = useMotionValue(0);
  const y = useMotionValue(0);
  const springX = useSpring(x, { stiffness: 300, damping: 20 });
  const springY = useSpring(y, { stiffness: 300, damping: 20 });

  const handleMouse = (e: React.MouseEvent) => {
    const rect = ref.current?.getBoundingClientRect();
    if (!rect) return;
    const centerX = rect.left + rect.width / 2;
    const centerY = rect.top + rect.height / 2;
    x.set((e.clientX - centerX) * 0.3);
    y.set((e.clientY - centerY) * 0.3);
  };

  const reset = () => { x.set(0); y.set(0); };

  return (
    <motion.button
      ref={ref}
      style={{ x: springX, y: springY }}
      onMouseMove={handleMouse}
      onMouseLeave={reset}
      className="px-8 py-3 rounded-full bg-[var(--accent)] text-white font-medium"
    >
      {children}
    </motion.button>
  );
}
```

### Smooth Page Transitions

```tsx
"use client";

import { AnimatePresence, motion } from "framer-motion";
import { usePathname } from "next/navigation";

export function PageTransition({ children }: { children: React.ReactNode }) {
  const pathname = usePathname();

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={pathname}
        initial={{ opacity: 0, y: 20 }}
        animate={{ opacity: 1, y: 0 }}
        exit={{ opacity: 0, y: -20 }}
        transition={{ duration: 0.3, ease: [0.25, 1, 0.5, 1] }}
      >
        {children}
      </motion.div>
    </AnimatePresence>
  );
}
```

### Card Hover with 3D Perspective

```tsx
"use client";

import { motion, useMotionValue, useTransform } from "framer-motion";

export function TiltCard({ children }: { children: React.ReactNode }) {
  const x = useMotionValue(0.5);
  const y = useMotionValue(0.5);
  const rotateX = useTransform(y, [0, 1], [8, -8]);
  const rotateY = useTransform(x, [0, 1], [-8, 8]);

  const handleMouse = (e: React.MouseEvent<HTMLDivElement>) => {
    const rect = e.currentTarget.getBoundingClientRect();
    x.set((e.clientX - rect.left) / rect.width);
    y.set((e.clientY - rect.top) / rect.height);
  };

  return (
    <motion.div
      onMouseMove={handleMouse}
      onMouseLeave={() => { x.set(0.5); y.set(0.5); }}
      style={{ rotateX, rotateY, transformPerspective: 800 }}
      transition={{ type: "spring", stiffness: 300, damping: 30 }}
      className="rounded-2xl bg-[var(--surface-2)] p-8 border border-[var(--border)]"
    >
      {children}
    </motion.div>
  );
}
```

### Scroll-Linked Animation (useScroll + useTransform)

```tsx
"use client";

import { motion, useScroll, useTransform } from "framer-motion";
import { useRef } from "react";

export function ScrollRevealSection() {
  const ref = useRef(null);
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ["start end", "end start"],
  });
  const opacity = useTransform(scrollYProgress, [0, 0.3, 0.7, 1], [0, 1, 1, 0]);
  const y = useTransform(scrollYProgress, [0, 0.3, 0.7, 1], [80, 0, 0, -80]);
  const scale = useTransform(scrollYProgress, [0, 0.3, 0.7, 1], [0.95, 1, 1, 0.95]);

  return (
    <motion.section ref={ref} style={{ opacity, y, scale }} className="py-24 px-6">
      <h2>Scroll-linked content</h2>
    </motion.section>
  );
}
```

### Spring Physics (useSpring)

```tsx
"use client";

import { motion, useSpring, useMotionValue } from "framer-motion";

export function SpringToggle() {
  const isOn = useMotionValue(0);
  const x = useSpring(isOn, { stiffness: 500, damping: 30, mass: 0.5 });

  return (
    <div
      className="w-16 h-8 bg-[var(--surface-3)] rounded-full p-1 cursor-pointer"
      onClick={() => isOn.set(isOn.get() === 0 ? 32 : 0)}
    >
      <motion.div style={{ x }} className="w-6 h-6 bg-[var(--accent)] rounded-full" />
    </div>
  );
}
```

---

## 8. GSAP Scroll Patterns

### ScrollTrigger Timeline with Pinning and Snapping

```tsx
"use client";

import { useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

gsap.registerPlugin(ScrollTrigger);

export function PinnedTimeline() {
  const containerRef = useRef<HTMLDivElement>(null);

  useGSAP(() => {
    const panels = gsap.utils.toArray<HTMLElement>(".panel");
    gsap.to(panels, {
      xPercent: -100 * (panels.length - 1),
      ease: "none",
      scrollTrigger: {
        trigger: containerRef.current,
        pin: true,
        scrub: 1,
        snap: 1 / (panels.length - 1),
        end: () => "+=" + (containerRef.current?.offsetWidth ?? 0),
      },
    });
  }, { scope: containerRef });

  return (
    <div ref={containerRef} className="flex overflow-hidden w-screen h-screen">
      {[1, 2, 3, 4].map((i) => (
        <div key={i} className="panel w-screen h-screen shrink-0 flex items-center justify-center">
          Panel {i}
        </div>
      ))}
    </div>
  );
}
```

### SplitText Character Animation

```ts
// Requires GSAP Club — or use a free split-text alternative
import { SplitText } from "gsap/SplitText";
gsap.registerPlugin(SplitText);

const split = new SplitText(".reveal-text", { type: "chars,words" });
gsap.from(split.chars, {
  opacity: 0,
  y: 50,
  rotateX: -90,
  stagger: 0.02,
  duration: 0.8,
  ease: "back.out(1.7)",
  scrollTrigger: {
    trigger: ".reveal-text",
    start: "top 80%",
    end: "top 20%",
    toggleActions: "play none none reverse",
  },
});
```

### Parallax Layers (different speeds)

```ts
gsap.to(".parallax-bg", {
  yPercent: -30,
  ease: "none",
  scrollTrigger: { trigger: ".parallax-section", start: "top bottom", end: "bottom top", scrub: true },
});

gsap.to(".parallax-mid", {
  yPercent: -50,
  ease: "none",
  scrollTrigger: { trigger: ".parallax-section", start: "top bottom", end: "bottom top", scrub: true },
});

gsap.to(".parallax-fg", {
  yPercent: -70,
  ease: "none",
  scrollTrigger: { trigger: ".parallax-section", start: "top bottom", end: "bottom top", scrub: true },
});
```

### Number Counter Animation

```tsx
"use client";

import { useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

gsap.registerPlugin(ScrollTrigger);

export function Counter({ target, suffix = "" }: { target: number; suffix?: string }) {
  const ref = useRef<HTMLSpanElement>(null);
  const numRef = useRef({ val: 0 });

  useGSAP(() => {
    gsap.to(numRef.current, {
      val: target,
      duration: 2,
      ease: "power2.out",
      scrollTrigger: { trigger: ref.current, start: "top 80%" },
      onUpdate: () => {
        if (ref.current) ref.current.textContent = Math.round(numRef.current.val) + suffix;
      },
    });
  });

  return <span ref={ref} className="text-5xl font-bold tabular-nums">0{suffix}</span>;
}
```

### Text Reveal Line-by-Line

```tsx
"use client";

import { useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

gsap.registerPlugin(ScrollTrigger);

export function TextReveal({ text }: { text: string }) {
  const containerRef = useRef<HTMLDivElement>(null);

  useGSAP(() => {
    const lines = gsap.utils.toArray<HTMLElement>(".reveal-line");
    lines.forEach((line) => {
      gsap.from(line, {
        clipPath: "inset(0 100% 0 0)",
        duration: 0.8,
        ease: "power4.inOut",
        scrollTrigger: { trigger: line, start: "top 85%" },
      });
    });
  }, { scope: containerRef });

  return (
    <div ref={containerRef}>
      {text.split("\n").map((line, i) => (
        <div key={i} className="reveal-line text-3xl font-bold overflow-hidden">
          {line}
        </div>
      ))}
    </div>
  );
}
```

---

## 9. Modern CSS Techniques

### Container Queries (95% support)

```css
.card-container {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card-inner { flex-direction: row; }
  .card-image { width: 40%; }
}

@container card (max-width: 399px) {
  .card-inner { flex-direction: column; }
  .card-image { width: 100%; aspect-ratio: 16/9; }
}
```

### :has() Selector (93%)

```css
/* Highlight form group when input is focused */
.form-group:has(input:focus) {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px color-mix(in oklch, var(--accent), transparent 80%);
}

/* Style card differently when it contains an image */
.card:has(img) { padding: 0; }
.card:not(:has(img)) { padding: 2rem; }

/* Form validation without JS */
.form-group:has(input:invalid:not(:placeholder-shown)) .error-msg {
  display: block;
}
```

### Scroll-Driven Animations (85%)

```css
/* Reading progress bar — zero JS */
.progress-bar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 3px;
  background: var(--accent);
  transform-origin: left;
  animation: grow-progress linear;
  animation-timeline: scroll(root);
}

@keyframes grow-progress {
  from { transform: scaleX(0); }
  to { transform: scaleX(1); }
}

/* Fade in on scroll — per element */
.fade-in-scroll {
  animation: fade-in linear both;
  animation-timeline: view();
  animation-range: entry 0% entry 100%;
}

@keyframes fade-in {
  from { opacity: 0; transform: translateY(30px); }
  to { opacity: 1; transform: translateY(0); }
}
```

### View Transitions API (83%)

```css
::view-transition-old(root) {
  animation: vt-fade-out 0.2s ease-out;
}
::view-transition-new(root) {
  animation: vt-fade-in 0.3s ease-in;
}

@keyframes vt-fade-out { to { opacity: 0; } }
@keyframes vt-fade-in { from { opacity: 0; } }
```

```tsx
// Use in Next.js with Link navigation
import { useRouter } from "next/navigation";

function navigate(href: string) {
  const router = useRouter();
  if (!document.startViewTransition) {
    router.push(href);
    return;
  }
  document.startViewTransition(() => router.push(href));
}
```

### Subgrid (93%)

```css
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.5rem;
}

.card-grid > .card {
  display: grid;
  grid-template-rows: subgrid;
  grid-row: span 3; /* header, body, footer aligned across cards */
}
```

### @property for Animated Gradients

```css
@property --gradient-angle {
  syntax: "<angle>";
  initial-value: 0deg;
  inherits: false;
}

.gradient-rotate {
  background: conic-gradient(from var(--gradient-angle), var(--brand-500), var(--brand-300), var(--brand-500));
  animation: rotate-gradient 3s linear infinite;
}

@keyframes rotate-gradient {
  to { --gradient-angle: 360deg; }
}
```

### CSS Masonry (2026)

```css
.masonry {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  grid-template-rows: masonry;
  gap: 1rem;
}
```

### @supports Progressive Enhancement

```css
/* Base experience */
.layout { display: flex; flex-wrap: wrap; gap: 1rem; }

@supports (container-type: inline-size) {
  .layout { container-type: inline-size; }
}

@supports (animation-timeline: scroll()) {
  .hero-element {
    animation: parallax linear;
    animation-timeline: scroll();
  }
}
```

---

## 10. Component Recipes

### Glassmorphism Card

```tsx
<div className="relative rounded-2xl border border-white/10 bg-white/5 backdrop-blur-xl p-8 overflow-hidden">
  <div className="absolute inset-0 bg-gradient-to-br from-white/10 to-transparent pointer-events-none" />
  <div className="relative z-10">{/* Content */}</div>
</div>
```

### Gradient Border (::before technique)

```css
.gradient-border {
  position: relative;
  border-radius: 1rem;
  background: var(--surface-1);
}

.gradient-border::before {
  content: "";
  position: absolute;
  inset: 0;
  padding: 1px;
  border-radius: inherit;
  background: linear-gradient(135deg, var(--brand-400), var(--brand-600));
  -webkit-mask: linear-gradient(#fff 0 0) content-box, linear-gradient(#fff 0 0);
  -webkit-mask-composite: xor;
  mask-composite: exclude;
  pointer-events: none;
}
```

### Spotlight Effect (cursor-following glow)

```tsx
"use client";

import { useRef } from "react";
import { motion, useMotionValue } from "framer-motion";

export function SpotlightCard({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const mouseX = useMotionValue(0);
  const mouseY = useMotionValue(0);

  const handleMouse = (e: React.MouseEvent) => {
    const rect = ref.current?.getBoundingClientRect();
    if (!rect) return;
    mouseX.set(e.clientX - rect.left);
    mouseY.set(e.clientY - rect.top);
  };

  return (
    <div
      ref={ref}
      onMouseMove={handleMouse}
      className="relative rounded-2xl border border-[var(--border)] bg-[var(--surface-2)] p-8 overflow-hidden"
    >
      <motion.div
        className="absolute w-64 h-64 rounded-full pointer-events-none -translate-x-1/2 -translate-y-1/2"
        style={{
          x: mouseX,
          y: mouseY,
          background: "radial-gradient(circle, oklch(0.7 0.15 264 / 0.15), transparent 70%)",
        }}
      />
      <div className="relative z-10">{children}</div>
    </div>
  );
}
```

### Custom Scrollbar + Selection Styling

```css
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: var(--surface-1); }
::-webkit-scrollbar-thumb { background: var(--neutral-400); border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: var(--neutral-500); }

html { scrollbar-color: var(--neutral-400) var(--surface-1); scrollbar-width: thin; }

::selection {
  background: color-mix(in oklch, var(--accent), transparent 70%);
  color: var(--text);
}
```

---

## 11. Hero Section Variants

### 1. Product-Led Hero

```tsx
<section className="relative min-h-screen flex items-center px-6 py-24 overflow-hidden">
  <div className="max-w-7xl mx-auto grid grid-cols-1 lg:grid-cols-[2fr_3fr] gap-16 items-center">
    <div className="space-y-8">
      <span className="font-mono text-xs tracking-widest uppercase text-[var(--accent)]">
        // Introducing v2.0
      </span>
      <h1 className="text-[var(--text-5xl)] font-[var(--font-display)] tracking-[-0.03em] leading-[0.95]">
        <span className="font-light text-[var(--text-muted)]">Ship </span>
        <span className="font-bold">10x faster</span>
      </h1>
      <p className="text-lg text-[var(--text-muted)] max-w-md">Description here.</p>
      <div className="flex gap-4">
        <button className="px-8 py-3 rounded-full bg-[var(--accent)] text-white font-medium">
          Get Started
        </button>
        <button className="px-8 py-3 rounded-full border border-[var(--border)] font-medium">
          Learn More
        </button>
      </div>
    </div>
    <div className="relative">
      <div className="absolute -inset-4 bg-[var(--accent)] opacity-10 blur-3xl rounded-full" />
      <div className="relative rounded-2xl overflow-hidden border border-[var(--border)]
                      [transform:perspective(1200px)_rotateY(-8deg)_rotateX(4deg)]
                      hover:[transform:perspective(1200px)_rotateY(0deg)_rotateX(0deg)]
                      transition-transform duration-700">
        <Image src="/product-screenshot.webp" alt="" width={1200} height={800} />
      </div>
    </div>
  </div>
</section>
```

### 2. Text-First Hero

```tsx
<section className="min-h-screen flex flex-col justify-center items-start px-6 py-24 max-w-5xl mx-auto">
  <span className="font-mono text-sm text-[var(--text-muted)] mb-8">Est. 2024</span>
  <h1 className="text-[var(--text-5xl)] font-[var(--font-display)] tracking-[-0.04em] leading-[0.9]">
    <span className="font-extralight">We create </span>
    <span className="font-black italic">digital experiences</span>
    <span className="font-extralight"> that </span>
    <span className="font-black italic">matter.</span>
  </h1>
  <div className="mt-16 flex items-center gap-8">
    <button className="px-10 py-4 bg-[var(--text)] text-[var(--bg)] rounded-full font-medium">
      Start a project
    </button>
    <span className="text-sm text-[var(--text-muted)]">Scroll to explore</span>
  </div>
</section>
```

### 3. Split Screen Hero

```tsx
<section className="min-h-screen grid grid-cols-1 lg:grid-cols-[2fr_3fr]">
  <div className="flex flex-col justify-center px-8 lg:px-16 py-24 bg-[var(--surface-2)]">
    <h1 className="text-[var(--text-4xl)] font-bold tracking-tight">Your headline here</h1>
    <p className="mt-4 text-[var(--text-muted)]">Supporting copy.</p>
    <button className="mt-8 self-start px-8 py-3 rounded-full bg-[var(--accent)] text-white">
      CTA
    </button>
  </div>
  <div className="relative overflow-hidden">
    {/* Lottie, video, or animated illustration */}
    <video autoPlay muted loop playsInline className="absolute inset-0 w-full h-full object-cover">
      <source src="/hero-anim.mp4" type="video/mp4" />
    </video>
  </div>
</section>
```

### 4. Video Background Hero

```tsx
<section className="relative min-h-screen flex items-center justify-center overflow-hidden">
  <video autoPlay muted loop playsInline className="absolute inset-0 w-full h-full object-cover">
    <source src="/hero-bg.mp4" type="video/mp4" />
  </video>
  <div className="absolute inset-0 bg-[var(--neutral-900)]/70" />
  <div className="relative z-10 text-center text-white space-y-6 px-6">
    <h1 className="text-[var(--text-5xl)] font-[var(--font-display)] font-bold tracking-tight">
      Bold Statement
    </h1>
    <p className="text-xl opacity-80 max-w-2xl mx-auto">Supporting copy here.</p>
  </div>
</section>
```

### 5. Interactive Hero (cursor-responsive)

```tsx
"use client";

import { motion, useMotionValue, useTransform } from "framer-motion";

export function InteractiveHero() {
  const mouseX = useMotionValue(0.5);
  const mouseY = useMotionValue(0.5);
  const rotateX = useTransform(mouseY, [0, 1], [5, -5]);
  const rotateY = useTransform(mouseX, [0, 1], [-5, 5]);

  return (
    <section
      className="min-h-screen flex items-center justify-center"
      onMouseMove={(e) => {
        mouseX.set(e.clientX / window.innerWidth);
        mouseY.set(e.clientY / window.innerHeight);
      }}
    >
      <motion.div
        style={{ rotateX, rotateY, transformPerspective: 1000 }}
        className="text-center"
      >
        <h1 className="text-[var(--text-5xl)] font-bold">Interactive</h1>
      </motion.div>
    </section>
  );
}
```

---

## 12. Navigation Patterns

### Full-Screen Overlay (clip-path circle reveal)

```css
.nav-overlay {
  position: fixed;
  inset: 0;
  z-index: 50;
  background: var(--neutral-900);
  clip-path: circle(0% at calc(100% - 2.5rem) 2.5rem);
  transition: clip-path 0.6s cubic-bezier(0.77, 0, 0.18, 1);
}

.nav-overlay[data-open="true"] {
  clip-path: circle(150% at calc(100% - 2.5rem) 2.5rem);
}
```

### Floating Pill Nav (blurred background, centered)

```tsx
<nav className="fixed top-6 left-1/2 -translate-x-1/2 z-50
                flex items-center gap-1 px-2 py-2 rounded-full
                bg-[var(--surface-1)]/80 backdrop-blur-xl border border-[var(--border)]
                shadow-lg">
  {links.map((link) => (
    <a key={link.href} href={link.href}
       className="px-5 py-2 rounded-full text-sm font-medium
                  hover:bg-[var(--surface-3)] transition-colors">
      {link.label}
    </a>
  ))}
</nav>
```

### Morphing Header (shrinks on scroll)

```tsx
"use client";

import { useScroll, useTransform, motion } from "framer-motion";

export function MorphingHeader() {
  const { scrollY } = useScroll();
  const height = useTransform(scrollY, [0, 100], [80, 56]);
  const opacity = useTransform(scrollY, [0, 50], [0, 1]);

  return (
    <motion.header
      style={{ height }}
      className="fixed top-0 inset-x-0 z-50 flex items-center px-6
                 backdrop-blur-xl border-b border-[var(--border)]"
    >
      <span className="font-bold text-lg">Logo</span>
      <nav className="ml-auto flex gap-6 text-sm">
        <a href="#">Features</a>
        <a href="#">Pricing</a>
        <a href="#">Docs</a>
      </nav>
    </motion.header>
  );
}
```

### Bottom Sheet Navigation (mobile)

```css
.bottom-sheet-nav {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 50;
  background: var(--surface-1);
  border-top: 1px solid var(--border);
  padding: 0.5rem 1rem calc(0.5rem + env(safe-area-inset-bottom));
  display: flex;
  justify-content: space-around;
}

@media (min-width: 768px) {
  .bottom-sheet-nav { display: none; }
}
```

---

## 13. Section Designs

### Pricing Table with Highlighted Plan

```tsx
<div className="grid grid-cols-1 md:grid-cols-3 gap-6 max-w-5xl mx-auto px-6">
  {plans.map((plan) => (
    <div key={plan.name} className={`rounded-2xl p-8 border ${
      plan.featured
        ? "border-[var(--accent)] bg-[var(--accent)]/5 ring-1 ring-[var(--accent)] scale-105"
        : "border-[var(--border)] bg-[var(--surface-2)]"
    }`}>
      {plan.featured && (
        <span className="inline-block px-3 py-1 text-xs font-medium rounded-full
                         bg-[var(--accent)] text-white mb-4">Most Popular</span>
      )}
      <h3 className="text-xl font-bold">{plan.name}</h3>
      <p className="text-4xl font-bold mt-4">
        {plan.price}<span className="text-base font-normal text-[var(--text-muted)]">/mo</span>
      </p>
      <ul className="mt-6 space-y-3">
        {plan.features.map((f) => (
          <li key={f} className="flex items-center gap-2 text-sm">
            <span className="text-[var(--accent)]">&#10003;</span> {f}
          </li>
        ))}
      </ul>
      <button className={`w-full mt-8 py-3 rounded-full font-medium ${
        plan.featured ? "bg-[var(--accent)] text-white" : "border border-[var(--border)]"
      }`}>
        Get Started
      </button>
    </div>
  ))}
</div>
```

### FAQ Accordion with Animations

```tsx
"use client";

import { motion, AnimatePresence } from "framer-motion";
import { useState } from "react";

export function FAQ({ items }: { items: { q: string; a: string }[] }) {
  const [open, setOpen] = useState<number | null>(null);

  return (
    <div className="max-w-2xl mx-auto divide-y divide-[var(--border)]">
      {items.map((item, i) => (
        <div key={i}>
          <button
            onClick={() => setOpen(open === i ? null : i)}
            className="w-full flex justify-between items-center py-5 text-left font-medium"
          >
            {item.q}
            <span className={`transition-transform duration-300 ${open === i ? "rotate-45" : ""}`}>+</span>
          </button>
          <AnimatePresence>
            {open === i && (
              <motion.div
                initial={{ height: 0, opacity: 0 }}
                animate={{ height: "auto", opacity: 1 }}
                exit={{ height: 0, opacity: 0 }}
                transition={{ duration: 0.3, ease: [0.16, 1, 0.3, 1] }}
                className="overflow-hidden"
              >
                <p className="pb-5 text-[var(--text-muted)]">{item.a}</p>
              </motion.div>
            )}
          </AnimatePresence>
        </div>
      ))}
    </div>
  );
}
```

### Stats Section with Animated Counters

```tsx
<section className="py-24 px-6 bg-[var(--surface-2)]">
  <div className="max-w-5xl mx-auto grid grid-cols-2 md:grid-cols-4 gap-8 text-center">
    <div>
      <Counter target={500} suffix="+" />
      <p className="mt-2 text-sm text-[var(--text-muted)]">Clients</p>
    </div>
    <div>
      <Counter target={98} suffix="%" />
      <p className="mt-2 text-sm text-[var(--text-muted)]">Satisfaction</p>
    </div>
    <div>
      <Counter target={15} suffix="M" />
      <p className="mt-2 text-sm text-[var(--text-muted)]">Revenue Generated</p>
    </div>
    <div>
      <Counter target={24} suffix="/7" />
      <p className="mt-2 text-sm text-[var(--text-muted)]">Support</p>
    </div>
  </div>
</section>
```

### Footer with Newsletter + Social

```tsx
<footer className="border-t border-[var(--border)] bg-[var(--surface-1)]">
  <div className="max-w-6xl mx-auto px-6 py-16 grid grid-cols-1 md:grid-cols-[2fr_1fr_1fr_1fr] gap-12">
    <div className="space-y-4">
      <h3 className="font-bold text-lg">Brand</h3>
      <p className="text-sm text-[var(--text-muted)] max-w-xs">Short tagline or description.</p>
      <form className="flex gap-2 mt-4">
        <input type="email" placeholder="Email" className="flex-1 px-4 py-2 rounded-full border border-[var(--border)] bg-transparent text-sm" />
        <button className="px-6 py-2 rounded-full bg-[var(--accent)] text-white text-sm font-medium">Join</button>
      </form>
    </div>
    <div>
      <h4 className="font-medium text-sm mb-4">Product</h4>
      <ul className="space-y-2 text-sm text-[var(--text-muted)]">
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Features</a></li>
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Pricing</a></li>
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Changelog</a></li>
      </ul>
    </div>
    <div>
      <h4 className="font-medium text-sm mb-4">Company</h4>
      <ul className="space-y-2 text-sm text-[var(--text-muted)]">
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">About</a></li>
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Blog</a></li>
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Careers</a></li>
      </ul>
    </div>
    <div>
      <h4 className="font-medium text-sm mb-4">Legal</h4>
      <ul className="space-y-2 text-sm text-[var(--text-muted)]">
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Privacy</a></li>
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Terms</a></li>
        <li><a href="#" className="hover:text-[var(--text)] transition-colors">Imprint</a></li>
      </ul>
    </div>
  </div>
</footer>
```

---

## 14. Magic UI / Aceternity UI Pattern Catalog

### Recommended Components by Category

| Category | Component | Best For | Library |
|---|---|---|---|
| **Backgrounds** | Dot Pattern, Grid Pattern | Landing pages, hero | Magic UI |
| **Backgrounds** | Spotlight, Aurora | SaaS hero sections | Aceternity |
| **Text Effects** | Text Generate, Typewriter | Hero headlines | Aceternity |
| **Text Effects** | Number Ticker, Animated Gradient Text | Stats, headings | Magic UI |
| **Cards** | 3D Card, Hover Border Gradient | Feature sections | Aceternity |
| **Cards** | Bento Grid | Dashboard-style features | Magic UI |
| **Navigation** | Floating Navbar, Sidebar | App shells | Aceternity |
| **Data Display** | Marquee, Orbit | Logos, integrations | Magic UI |
| **Data Display** | Infinite Moving Cards | Testimonials | Aceternity |
| **Interactive** | Globe, Particles | Hero backgrounds | Magic UI |
| **Interactive** | Tracing Beam, Sticky Scroll | Blog, docs | Aceternity |
| **Buttons** | Shimmer Button, Ripple | CTAs | Magic UI |

### Usage Guidelines

- **Landing pages**: Aurora/Spotlight background + Text Generate headline + Bento Grid features
- **Dashboards**: Bento Grid layout + Number Ticker stats + Marquee integrations
- **Docs/Blog**: Tracing Beam + Sticky Scroll Reveal
- **SaaS**: Floating Navbar + 3D Cards + Shimmer Button CTAs

---

## 15. Premium CSS Checklist

Apply all of these to every project:

```css
/* 1. Smooth scroll */
html { scroll-behavior: smooth; }

/* 2. Custom scrollbar */
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-thumb { background: var(--neutral-400); border-radius: 4px; }
html { scrollbar-color: var(--neutral-400) transparent; scrollbar-width: thin; }

/* 3. Selection */
::selection { background: color-mix(in oklch, var(--accent), transparent 70%); }

/* 4. Focus-visible (keyboard only) */
:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}
:focus:not(:focus-visible) { outline: none; }

/* 5. Spacing scale */
:root {
  --space-1: 0.25rem;  --space-2: 0.5rem;   --space-3: 0.75rem;
  --space-4: 1rem;     --space-6: 1.5rem;    --space-8: 2rem;
  --space-12: 3rem;    --space-16: 4rem;     --space-24: 6rem;
  --space-32: 8rem;
}

/* 6. Shadow scale */
:root {
  --shadow-sm: 0 1px 2px oklch(0 0 0 / 0.05);
  --shadow-md: 0 4px 12px oklch(0 0 0 / 0.08);
  --shadow-lg: 0 12px 32px oklch(0 0 0 / 0.12);
  --shadow-xl: 0 24px 48px oklch(0 0 0 / 0.16);
}

/* 7. Border radius scale */
:root {
  --radius-sm: 4px;    --radius-md: 8px;
  --radius-lg: 16px;   --radius-xl: 24px;
  --radius-full: 9999px;
}

/* 8. Motion curves — never use linear or ease */
:root {
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-out-quart: cubic-bezier(0.25, 1, 0.5, 1);
  --ease-in-out-quart: cubic-bezier(0.76, 0, 0.24, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
}

/* 9. Base typography reset */
body {
  font-family: var(--font-body);
  font-size: var(--text-base);
  line-height: 1.6;
  color: var(--text);
  background: var(--bg);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

h1, h2, h3, h4 {
  font-family: var(--font-display);
  line-height: 1.1;
  letter-spacing: -0.02em;
  text-wrap: balance;
}

p { text-wrap: pretty; max-inline-size: 65ch; }

/* 10. Reduced motion */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

---

## Quick Reference: Custom Bezier Curves

| Effect | Curve | Use For |
|---|---|---|
| Snappy exit | `cubic-bezier(0.16, 1, 0.3, 1)` | Elements leaving, reveals |
| Smooth entrance | `cubic-bezier(0.25, 1, 0.5, 1)` | Elements appearing |
| Dramatic | `cubic-bezier(0.76, 0, 0.24, 1)` | Page transitions |
| Bouncy | `cubic-bezier(0.34, 1.56, 0.64, 1)` | Buttons, toggles |
| Heavy | `cubic-bezier(0.7, 0, 0.3, 1)` | Overlays, modals |
