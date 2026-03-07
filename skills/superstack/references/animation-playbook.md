# Animation Playbook — Superstack Reference

## Table of Contents
- [1. Framer Motion Basics](#1-framer-motion-basics)
  - [Motion Components](#motion-components)
  - [AnimatePresence (Exit Animations)](#animatepresence-exit-animations)
  - [Layout Animations](#layout-animations)
- [2. Scroll Animations](#2-scroll-animations)
  - [useScroll + useTransform](#usescroll--usetransform)
  - [Scroll-linked Progress Bar](#scroll-linked-progress-bar)
  - [Reveal on Scroll (useInView alternative)](#reveal-on-scroll-useinview-alternative)
- [3. GSAP with Lenis](#3-gsap-with-lenis)
  - [Setup](#setup)
  - [Lenis + GSAP ScrollTrigger Integration](#lenis--gsap-scrolltrigger-integration)
  - [Pin Section with Scrub](#pin-section-with-scrub)
  - [GSAP Cleanup Rule](#gsap-cleanup-rule)
- [4. Stagger Patterns](#4-stagger-patterns)
  - [Framer Motion staggerChildren](#framer-motion-staggerchildren)
  - [Custom Stagger with Delay Calculation](#custom-stagger-with-delay-calculation)
  - [GSAP Stagger](#gsap-stagger)
- [5. Page Transitions (App Router)](#5-page-transitions-app-router)
  - [Shared Layout Animation](#shared-layout-animation)
  - [View Transitions API (Native)](#view-transitions-api-native)
- [6. Micro-Interactions](#6-micro-interactions)
  - [Button Press (Scale)](#button-press-scale)
  - [Hover Card Effect](#hover-card-effect)
  - [Toggle Switch](#toggle-switch)
  - [Success Checkmark](#success-checkmark)
- [7. CSS-Only Animations](#7-css-only-animations)
  - [@keyframes](#keyframes)
  - [Scroll-Driven Animations (CSS)](#scroll-driven-animations-css)
  - [View Transitions API](#view-transitions-api)
- [8. Performance Rules](#8-performance-rules)
  - [The Golden Rules](#the-golden-rules)
  - [GPU Compositing](#gpu-compositing)
  - [Avoid Layout Thrashing](#avoid-layout-thrashing)
  - [requestAnimationFrame](#requestanimationframe)
  - [Framer Motion Performance Tips](#framer-motion-performance-tips)
- [9. Reduced Motion](#9-reduced-motion)
  - [CSS Media Query](#css-media-query)
  - [Framer Motion reducedMotion](#framer-motion-reducedmotion)
  - [Custom Hook](#custom-hook)
  - [Accessible Defaults](#accessible-defaults)
- [10. Magic UI / Aceternity Patterns](#10-magic-ui--aceternity-patterns)
  - [Marquee (Infinite Scroll)](#marquee-infinite-scroll)
  - [Spotlight Effect](#spotlight-effect)
  - [Aurora Background](#aurora-background)
  - [Meteors](#meteors)
  - [Particle Effect (Canvas)](#particle-effect-canvas)
  - [Implementation Tips](#implementation-tips)

## 1. Framer Motion Basics

### Motion Components

```tsx
import { motion, AnimatePresence } from "framer-motion";

// Basic animation
function FadeIn({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -20 }}
      transition={{ duration: 0.3, ease: "easeOut" }}
    >
      {children}
    </motion.div>
  );
}
```

### AnimatePresence (Exit Animations)

```tsx
"use client";

import { useState } from "react";
import { motion, AnimatePresence } from "framer-motion";

function Notification({ id, message, onDismiss }: NotificationProps) {
  return (
    <motion.div
      layout
      initial={{ opacity: 0, x: 100 }}
      animate={{ opacity: 1, x: 0 }}
      exit={{ opacity: 0, x: 100, transition: { duration: 0.2 } }}
      className="rounded-lg bg-white p-4 shadow-lg"
    >
      <p>{message}</p>
      <button onClick={() => onDismiss(id)}>Dismiss</button>
    </motion.div>
  );
}

function NotificationList() {
  const [notifications, setNotifications] = useState<
    { id: string; message: string }[]
  >([]);

  const dismiss = (id: string) =>
    setNotifications((n) => n.filter((x) => x.id !== id));

  return (
    <div className="fixed right-4 top-4 flex flex-col gap-2">
      <AnimatePresence mode="popLayout">
        {notifications.map((n) => (
          <Notification key={n.id} {...n} onDismiss={dismiss} />
        ))}
      </AnimatePresence>
    </div>
  );
}
```

### Layout Animations

```tsx
function ExpandableCard({ title, content }: { title: string; content: string }) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <motion.div
      layout
      onClick={() => setIsOpen(!isOpen)}
      className="cursor-pointer overflow-hidden rounded-xl bg-white p-4 shadow"
      style={{ borderRadius: "12px" }} // avoid border-radius glitch with layout
    >
      <motion.h3 layout="position" className="text-lg font-semibold">
        {title}
      </motion.h3>
      <AnimatePresence>
        {isOpen && (
          <motion.p
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            className="mt-2 text-gray-600"
          >
            {content}
          </motion.p>
        )}
      </AnimatePresence>
    </motion.div>
  );
}
```

---

## 2. Scroll Animations

### useScroll + useTransform

```tsx
"use client";

import { useRef } from "react";
import { motion, useScroll, useTransform } from "framer-motion";

function ParallaxHero() {
  const ref = useRef<HTMLDivElement>(null);

  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ["start start", "end start"], // when element enters/leaves
  });

  const y = useTransform(scrollYProgress, [0, 1], ["0%", "50%"]);
  const opacity = useTransform(scrollYProgress, [0, 0.8], [1, 0]);

  return (
    <div ref={ref} className="relative h-screen overflow-hidden">
      <motion.div
        style={{ y }}
        className="absolute inset-0 bg-cover bg-center"
      >
        <img src="/hero.jpg" alt="" className="h-full w-full object-cover" />
      </motion.div>
      <motion.div style={{ opacity }} className="relative z-10 flex h-full items-center justify-center">
        <h1 className="text-6xl font-bold text-white">Hero Title</h1>
      </motion.div>
    </div>
  );
}
```

### Scroll-linked Progress Bar

```tsx
function ScrollProgressBar() {
  const { scrollYProgress } = useScroll();

  return (
    <motion.div
      style={{ scaleX: scrollYProgress, transformOrigin: "left" }}
      className="fixed left-0 right-0 top-0 z-50 h-1 bg-blue-500"
    />
  );
}
```

### Reveal on Scroll (useInView alternative)

```tsx
import { motion, useInView } from "framer-motion";
import { useRef } from "react";

function RevealSection({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const isInView = useInView(ref, { once: true, margin: "-100px" });

  return (
    <div ref={ref}>
      <motion.div
        initial={{ opacity: 0, y: 60 }}
        animate={isInView ? { opacity: 1, y: 0 } : {}}
        transition={{ duration: 0.6, ease: "easeOut" }}
      >
        {children}
      </motion.div>
    </div>
  );
}
```

---

## 3. GSAP with Lenis

### Setup

```bash
bun add gsap @studio-freight/lenis
```

### Lenis + GSAP ScrollTrigger Integration

```tsx
"use client";

import { useEffect, useRef } from "react";
import Lenis from "@studio-freight/lenis";
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";

gsap.registerPlugin(ScrollTrigger);

export function SmoothScrollProvider({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    const lenis = new Lenis({
      duration: 1.2,
      easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
      smoothWheel: true,
    });

    // Sync Lenis with GSAP ScrollTrigger
    lenis.on("scroll", ScrollTrigger.update);

    gsap.ticker.add((time) => {
      lenis.raf(time * 1000);
    });
    gsap.ticker.lagSmoothing(0);

    return () => {
      lenis.destroy();
      gsap.ticker.remove(lenis.raf);
    };
  }, []);

  return <>{children}</>;
}
```

### Pin Section with Scrub

```tsx
"use client";

import { useEffect, useRef } from "react";
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";

gsap.registerPlugin(ScrollTrigger);

function PinnedSection() {
  const sectionRef = useRef<HTMLDivElement>(null);
  const contentRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const section = sectionRef.current;
    const content = contentRef.current;
    if (!section || !content) return;

    const ctx = gsap.context(() => {
      // Pin the section while scrolling through it
      gsap.to(content, {
        x: "-200%", // horizontal scroll effect
        ease: "none",
        scrollTrigger: {
          trigger: section,
          start: "top top",
          end: "+=3000", // scroll distance
          pin: true,
          scrub: 1, // smooth scrub
          anticipatePin: 1,
        },
      });
    }, section);

    return () => ctx.revert(); // cleanup
  }, []);

  return (
    <div ref={sectionRef} className="overflow-hidden">
      <div ref={contentRef} className="flex w-[400%]">
        <div className="flex h-screen w-screen items-center justify-center bg-red-500">
          <h2 className="text-5xl text-white">Section 1</h2>
        </div>
        <div className="flex h-screen w-screen items-center justify-center bg-blue-500">
          <h2 className="text-5xl text-white">Section 2</h2>
        </div>
        <div className="flex h-screen w-screen items-center justify-center bg-green-500">
          <h2 className="text-5xl text-white">Section 3</h2>
        </div>
      </div>
    </div>
  );
}
```

### GSAP Cleanup Rule

Always use `gsap.context()` and call `.revert()` in the cleanup function. This ensures all animations and ScrollTriggers are properly removed.

---

## 4. Stagger Patterns

### Framer Motion staggerChildren

```tsx
const containerVariants = {
  hidden: { opacity: 0 },
  show: {
    opacity: 1,
    transition: {
      staggerChildren: 0.1,
      delayChildren: 0.2,
    },
  },
};

const itemVariants = {
  hidden: { opacity: 0, y: 30 },
  show: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.4, ease: "easeOut" },
  },
};

function FeatureGrid({ features }: { features: Feature[] }) {
  return (
    <motion.div
      variants={containerVariants}
      initial="hidden"
      whileInView="show"
      viewport={{ once: true, margin: "-50px" }}
      className="grid grid-cols-3 gap-6"
    >
      {features.map((feature) => (
        <motion.div
          key={feature.id}
          variants={itemVariants}
          className="rounded-lg border p-6"
        >
          <h3>{feature.title}</h3>
          <p>{feature.description}</p>
        </motion.div>
      ))}
    </motion.div>
  );
}
```

### Custom Stagger with Delay Calculation

```tsx
function StaggeredList({ items }: { items: string[] }) {
  return (
    <div>
      {items.map((item, i) => (
        <motion.div
          key={item}
          initial={{ opacity: 0, x: -40 }}
          animate={{ opacity: 1, x: 0 }}
          transition={{
            duration: 0.4,
            delay: i * 0.08, // 80ms between each
            ease: [0.25, 0.46, 0.45, 0.94], // custom easing
          }}
          className="border-b py-3"
        >
          {item}
        </motion.div>
      ))}
    </div>
  );
}
```

### GSAP Stagger

```tsx
useEffect(() => {
  const ctx = gsap.context(() => {
    gsap.from(".card", {
      y: 60,
      opacity: 0,
      duration: 0.6,
      stagger: {
        each: 0.1,
        from: "start", // "start" | "end" | "center" | "edges" | "random"
        grid: [3, 4], // for grid layouts
        axis: "y",
      },
      scrollTrigger: {
        trigger: ".cards-container",
        start: "top 80%",
      },
    });
  });

  return () => ctx.revert();
}, []);
```

---

## 5. Page Transitions (App Router)

### Shared Layout Animation

```tsx
// components/page-transition.tsx
"use client";

import { AnimatePresence, motion } from "framer-motion";
import { usePathname } from "next/navigation";

const pageVariants = {
  initial: { opacity: 0, y: 8 },
  enter: { opacity: 1, y: 0, transition: { duration: 0.3, ease: "easeOut" } },
  exit: { opacity: 0, y: -8, transition: { duration: 0.2 } },
};

export function PageTransition({ children }: { children: React.ReactNode }) {
  const pathname = usePathname();

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={pathname}
        variants={pageVariants}
        initial="initial"
        animate="enter"
        exit="exit"
      >
        {children}
      </motion.div>
    </AnimatePresence>
  );
}
```

```tsx
// app/layout.tsx
import { PageTransition } from "@/components/page-transition";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Navbar />
        <PageTransition>{children}</PageTransition>
      </body>
    </html>
  );
}
```

**Caveat:** AnimatePresence `mode="wait"` with App Router can be tricky because Next.js doesn't unmount the old page before mounting the new one. For reliable exit animations, consider using `next-view-transitions` or the View Transitions API.

### View Transitions API (Native)

```tsx
"use client";

import { useRouter } from "next/navigation";

function navigateWithTransition(router: ReturnType<typeof useRouter>, href: string) {
  if (!document.startViewTransition) {
    router.push(href);
    return;
  }

  document.startViewTransition(() => {
    router.push(href);
  });
}
```

```css
/* Global CSS */
::view-transition-old(root) {
  animation: fade-out 0.2s ease-in;
}

::view-transition-new(root) {
  animation: fade-in 0.3s ease-out;
}

@keyframes fade-out {
  to { opacity: 0; }
}

@keyframes fade-in {
  from { opacity: 0; }
}
```

---

## 6. Micro-Interactions

### Button Press (Scale)

```tsx
function PressableButton({ children, onClick }: { children: React.ReactNode; onClick: () => void }) {
  return (
    <motion.button
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.97 }}
      transition={{ type: "spring", stiffness: 400, damping: 17 }}
      onClick={onClick}
      className="rounded-lg bg-blue-600 px-6 py-3 text-white"
    >
      {children}
    </motion.button>
  );
}
```

### Hover Card Effect

```tsx
function HoverCard({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      whileHover={{
        y: -4,
        boxShadow: "0 20px 40px rgba(0, 0, 0, 0.1)",
      }}
      transition={{ type: "spring", stiffness: 300, damping: 20 }}
      className="rounded-xl border bg-white p-6"
    >
      {children}
    </motion.div>
  );
}
```

### Toggle Switch

```tsx
function Toggle({ isOn, onToggle }: { isOn: boolean; onToggle: () => void }) {
  return (
    <button
      onClick={onToggle}
      className={`flex h-8 w-14 items-center rounded-full p-1 ${
        isOn ? "bg-blue-600" : "bg-gray-300"
      }`}
    >
      <motion.div
        layout
        transition={{ type: "spring", stiffness: 500, damping: 30 }}
        className="h-6 w-6 rounded-full bg-white shadow"
        style={{ marginLeft: isOn ? "auto" : 0 }}
      />
    </button>
  );
}
```

### Success Checkmark

```tsx
const checkmarkVariants = {
  hidden: { pathLength: 0, opacity: 0 },
  visible: {
    pathLength: 1,
    opacity: 1,
    transition: { duration: 0.5, ease: "easeInOut", delay: 0.2 },
  },
};

const circleVariants = {
  hidden: { scale: 0 },
  visible: {
    scale: 1,
    transition: { type: "spring", stiffness: 200, damping: 15 },
  },
};

function SuccessCheck() {
  return (
    <motion.svg
      width="64"
      height="64"
      viewBox="0 0 64 64"
      initial="hidden"
      animate="visible"
    >
      <motion.circle
        cx="32"
        cy="32"
        r="30"
        fill="none"
        stroke="#22c55e"
        strokeWidth="3"
        variants={circleVariants}
      />
      <motion.path
        d="M20 33 L28 41 L44 25"
        fill="none"
        stroke="#22c55e"
        strokeWidth="3"
        strokeLinecap="round"
        strokeLinejoin="round"
        variants={checkmarkVariants}
      />
    </motion.svg>
  );
}
```

---

## 7. CSS-Only Animations

### @keyframes

```css
/* Pulse */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

/* Slide up */
@keyframes slide-up {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.animate-slide-up {
  animation: slide-up 0.4s ease-out forwards;
}

/* Spin */
@keyframes spin {
  to { transform: rotate(360deg); }
}

.animate-spin {
  animation: spin 1s linear infinite;
}
```

### Scroll-Driven Animations (CSS)

```css
/* Progress bar driven by scroll position */
.scroll-progress {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 4px;
  background: blue;
  transform-origin: left;
  animation: grow-progress linear;
  animation-timeline: scroll();
}

@keyframes grow-progress {
  from { transform: scaleX(0); }
  to { transform: scaleX(1); }
}

/* Reveal elements as they enter viewport */
.reveal-on-scroll {
  animation: reveal linear both;
  animation-timeline: view();
  animation-range: entry 0% entry 100%;
}

@keyframes reveal {
  from {
    opacity: 0;
    transform: translateY(40px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

### View Transitions API

```css
/* Named view transitions for shared element animations */
.product-image {
  view-transition-name: product-hero;
}

::view-transition-old(product-hero) {
  animation: 300ms ease-out both fade-out;
}

::view-transition-new(product-hero) {
  animation: 300ms ease-in both fade-in;
}
```

---

## 8. Performance Rules

### The Golden Rules

1. **Only animate `transform` and `opacity`** — these don't trigger layout or paint.
2. **Never animate `width`, `height`, `top`, `left`, `margin`, `padding`** — use `transform: translate/scale` instead.
3. **Use `will-change` sparingly** — only on elements about to animate, remove after.

### GPU Compositing

```css
/* Promote to GPU layer */
.animated-element {
  will-change: transform;
  /* or use the hack: */
  transform: translateZ(0);
}

/* Remove will-change after animation completes */
.animated-element.done {
  will-change: auto;
}
```

### Avoid Layout Thrashing

```tsx
// BAD — triggers layout recalc on every read
items.forEach((item) => {
  const height = item.offsetHeight; // READ (forces layout)
  item.style.height = height + 10 + "px"; // WRITE (invalidates layout)
});

// GOOD — batch reads, then writes
const heights = items.map((item) => item.offsetHeight); // all reads
items.forEach((item, i) => {
  item.style.height = heights[i] + 10 + "px"; // all writes
});
```

### requestAnimationFrame

```tsx
// Smooth custom animation loop
function animateValue(
  from: number,
  to: number,
  duration: number,
  onUpdate: (value: number) => void
) {
  const start = performance.now();

  function frame(now: number) {
    const elapsed = now - start;
    const progress = Math.min(elapsed / duration, 1);
    const eased = 1 - Math.pow(1 - progress, 3); // ease-out cubic
    const value = from + (to - from) * eased;

    onUpdate(value);

    if (progress < 1) {
      requestAnimationFrame(frame);
    }
  }

  requestAnimationFrame(frame);
}
```

### Framer Motion Performance Tips

```tsx
// Use layout="position" instead of layout={true} when only position changes
<motion.div layout="position" />

// Use layoutId for shared element transitions (more performant than layout)
<motion.div layoutId={`card-${id}`} />

// Disable layout animations on children that don't need it
<motion.div layout>
  <motion.p layout="preserve-aspect">Stable child</motion.p>
</motion.div>
```

---

## 9. Reduced Motion

### CSS Media Query

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

### Framer Motion reducedMotion

```tsx
// Global setting via MotionConfig
import { MotionConfig } from "framer-motion";

function App({ children }: { children: React.ReactNode }) {
  return (
    <MotionConfig reducedMotion="user">
      {children}
    </MotionConfig>
  );
}
// "user" = respects prefers-reduced-motion
// "always" = always reduce motion
// "never" = ignore the preference
```

### Custom Hook

```tsx
import { useEffect, useState } from "react";

function usePrefersReducedMotion() {
  const [prefersReduced, setPrefersReduced] = useState(false);

  useEffect(() => {
    const mq = window.matchMedia("(prefers-reduced-motion: reduce)");
    setPrefersReduced(mq.matches);

    const handler = (e: MediaQueryListEvent) => setPrefersReduced(e.matches);
    mq.addEventListener("change", handler);
    return () => mq.removeEventListener("change", handler);
  }, []);

  return prefersReduced;
}

// Usage
function FancySection() {
  const reducedMotion = usePrefersReducedMotion();

  return (
    <motion.div
      initial={{ opacity: 0, y: reducedMotion ? 0 : 40 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: reducedMotion ? 0 : 0.6 }}
    >
      Content
    </motion.div>
  );
}
```

### Accessible Defaults

- Always provide `prefers-reduced-motion` handling.
- Decorative animations (particles, aurora, etc.) should be disabled entirely for reduced motion.
- Functional animations (page transitions, accordions) should be instant (0ms duration), not removed.
- Use `MotionConfig reducedMotion="user"` at the app root.

---

## 10. Magic UI / Aceternity Patterns

### Marquee (Infinite Scroll)

```tsx
interface MarqueeProps {
  children: React.ReactNode;
  speed?: number; // pixels per second
  direction?: "left" | "right";
  pauseOnHover?: boolean;
}

function Marquee({
  children,
  speed = 40,
  direction = "left",
  pauseOnHover = true,
}: MarqueeProps) {
  return (
    <div
      className="group flex overflow-hidden"
      style={{ "--speed": `${speed}s` } as React.CSSProperties}
    >
      {[0, 1].map((i) => (
        <div
          key={i}
          className={`flex shrink-0 gap-4 ${
            pauseOnHover ? "group-hover:[animation-play-state:paused]" : ""
          }`}
          style={{
            animation: `marquee var(--speed) linear infinite`,
            animationDirection: direction === "right" ? "reverse" : "normal",
          }}
        >
          {children}
        </div>
      ))}
    </div>
  );
}
```

```css
@keyframes marquee {
  from { transform: translateX(0); }
  to { transform: translateX(-100%); }
}
```

### Spotlight Effect

```tsx
"use client";

import { useRef, useState } from "react";

function SpotlightCard({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e: React.MouseEvent) => {
    const rect = ref.current?.getBoundingClientRect();
    if (!rect) return;
    setPosition({
      x: e.clientX - rect.left,
      y: e.clientY - rect.top,
    });
  };

  return (
    <div
      ref={ref}
      onMouseMove={handleMouseMove}
      className="group relative overflow-hidden rounded-xl border border-white/10 bg-gray-900 p-8"
    >
      {/* Spotlight gradient */}
      <div
        className="pointer-events-none absolute -inset-px opacity-0 transition-opacity duration-300 group-hover:opacity-100"
        style={{
          background: `radial-gradient(600px circle at ${position.x}px ${position.y}px, rgba(59, 130, 246, 0.15), transparent 40%)`,
        }}
      />
      <div className="relative z-10">{children}</div>
    </div>
  );
}
```

### Aurora Background

```tsx
function AuroraBackground({ children }: { children: React.ReactNode }) {
  return (
    <div className="relative min-h-screen overflow-hidden bg-zinc-950">
      {/* Aurora blobs */}
      <div className="absolute inset-0">
        <div className="aurora-blob aurora-blob-1" />
        <div className="aurora-blob aurora-blob-2" />
        <div className="aurora-blob aurora-blob-3" />
      </div>
      <div className="relative z-10">{children}</div>
    </div>
  );
}
```

```css
.aurora-blob {
  position: absolute;
  width: 40%;
  height: 40%;
  border-radius: 50%;
  filter: blur(100px);
  opacity: 0.3;
}

.aurora-blob-1 {
  top: 10%;
  left: 20%;
  background: #3b82f6;
  animation: aurora-drift 8s ease-in-out infinite alternate;
}

.aurora-blob-2 {
  top: 40%;
  right: 20%;
  background: #8b5cf6;
  animation: aurora-drift 10s ease-in-out infinite alternate-reverse;
}

.aurora-blob-3 {
  bottom: 10%;
  left: 40%;
  background: #06b6d4;
  animation: aurora-drift 12s ease-in-out infinite alternate;
}

@keyframes aurora-drift {
  0% { transform: translate(0, 0) scale(1); }
  33% { transform: translate(30px, -20px) scale(1.1); }
  66% { transform: translate(-20px, 20px) scale(0.9); }
  100% { transform: translate(10px, -10px) scale(1.05); }
}
```

### Meteors

```tsx
function Meteors({ count = 20 }: { count?: number }) {
  const meteors = Array.from({ length: count }, (_, i) => ({
    id: i,
    left: `${Math.random() * 100}%`,
    delay: `${Math.random() * 5}s`,
    duration: `${Math.random() * 2 + 1}s`,
  }));

  return (
    <div className="pointer-events-none absolute inset-0 overflow-hidden">
      {meteors.map((m) => (
        <div
          key={m.id}
          className="meteor"
          style={{
            left: m.left,
            animationDelay: m.delay,
            animationDuration: m.duration,
          }}
        />
      ))}
    </div>
  );
}
```

```css
.meteor {
  position: absolute;
  top: -20px;
  width: 2px;
  height: 20px;
  background: linear-gradient(to bottom, rgba(255, 255, 255, 0.8), transparent);
  border-radius: 9999px;
  animation: meteor-fall linear infinite;
  box-shadow: 0 0 6px rgba(255, 255, 255, 0.3);
}

@keyframes meteor-fall {
  0% {
    transform: translateY(0) rotate(-45deg);
    opacity: 1;
  }
  70% {
    opacity: 1;
  }
  100% {
    transform: translateY(100vh) rotate(-45deg);
    opacity: 0;
  }
}
```

### Particle Effect (Canvas)

```tsx
"use client";

import { useEffect, useRef } from "react";

interface Particle {
  x: number;
  y: number;
  vx: number;
  vy: number;
  size: number;
  alpha: number;
}

function ParticleField({ count = 80 }: { count?: number }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext("2d")!;
    let animationId: number;

    const resize = () => {
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
    };
    resize();
    window.addEventListener("resize", resize);

    const particles: Particle[] = Array.from({ length: count }, () => ({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      vx: (Math.random() - 0.5) * 0.5,
      vy: (Math.random() - 0.5) * 0.5,
      size: Math.random() * 2 + 0.5,
      alpha: Math.random() * 0.5 + 0.2,
    }));

    function draw() {
      ctx.clearRect(0, 0, canvas!.width, canvas!.height);

      particles.forEach((p) => {
        p.x += p.vx;
        p.y += p.vy;

        // Wrap around edges
        if (p.x < 0) p.x = canvas!.width;
        if (p.x > canvas!.width) p.x = 0;
        if (p.y < 0) p.y = canvas!.height;
        if (p.y > canvas!.height) p.y = 0;

        ctx.beginPath();
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        ctx.fillStyle = `rgba(255, 255, 255, ${p.alpha})`;
        ctx.fill();
      });

      // Draw connections
      for (let i = 0; i < particles.length; i++) {
        for (let j = i + 1; j < particles.length; j++) {
          const dx = particles[i].x - particles[j].x;
          const dy = particles[i].y - particles[j].y;
          const dist = Math.sqrt(dx * dx + dy * dy);

          if (dist < 120) {
            ctx.beginPath();
            ctx.moveTo(particles[i].x, particles[i].y);
            ctx.lineTo(particles[j].x, particles[j].y);
            ctx.strokeStyle = `rgba(255, 255, 255, ${0.15 * (1 - dist / 120)})`;
            ctx.lineWidth = 0.5;
            ctx.stroke();
          }
        }
      }

      animationId = requestAnimationFrame(draw);
    }

    draw();

    return () => {
      cancelAnimationFrame(animationId);
      window.removeEventListener("resize", resize);
    };
  }, [count]);

  return (
    <canvas
      ref={canvasRef}
      className="pointer-events-none absolute inset-0"
    />
  );
}
```

### Implementation Tips

1. **Performance:** All decorative effects (aurora, particles, meteors) should use `pointer-events-none` and be placed behind content with `z-index`.
2. **Canvas vs CSS:** Use Canvas for particles and complex physics. Use CSS animations for simpler effects (aurora, meteors).
3. **Reduced motion:** Wrap all decorative effects in a reduced-motion check — disable them entirely for users who prefer reduced motion.
4. **Mobile:** Reduce particle count and blur radius on mobile. Use `matchMedia` or a Tailwind `md:` breakpoint check.
5. **Bundle size:** Import only what you need from Magic UI / Aceternity. Copy the component source rather than importing the full package.
