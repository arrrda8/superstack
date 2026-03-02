# Design Patterns Reference — Superstack

This document contains specific, implementable patterns for creating award-winning web design. Read this when building the front-end to ensure the result doesn't look like generic AI output.

## Table of Contents
1. Layout Patterns
2. Typography Techniques
3. Framer Motion Recipes
4. GSAP Scroll Patterns
5. Component Recipes
6. Color & Theming
7. Responsive Strategies

---

## 1. Layout Patterns

### The Asymmetric Split
Instead of 50/50 columns, use 40/60 or 35/65 splits. Let one side breathe. This immediately breaks the "AI template" look.

```css
.hero-split {
  display: grid;
  grid-template-columns: 2fr 3fr;
  gap: clamp(2rem, 5vw, 6rem);
  align-items: center;
}
```

### The Overlapping Grid
Elements that overlap their grid cells create depth and visual interest. Use negative margins or CSS Grid with named areas.

```css
.feature-card {
  position: relative;
  z-index: 1;
}
.feature-image {
  margin-top: -4rem;
  margin-left: -2rem;
  position: relative;
  z-index: 2;
}
```

### The Full-Bleed Section
Alternate between contained (max-width) and full-bleed sections. The rhythm creates visual variety.

```css
.section-contained { max-width: 1200px; margin: 0 auto; padding: 0 2rem; }
.section-bleed { width: 100vw; position: relative; left: 50%; margin-left: -50vw; }
```

### The Bento Grid
Inspired by Apple — variable-size cards in a grid. Some cards span 2 columns, some are tall, some are small. Each card has a different visual treatment (gradient, image, icon, stat).

```css
.bento {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-auto-rows: minmax(200px, auto);
  gap: 1rem;
}
.bento-large { grid-column: span 2; grid-row: span 2; }
.bento-wide { grid-column: span 2; }
.bento-tall { grid-row: span 2; }
```

### The Horizontal Scroll Section
For showcasing multiple items (testimonials, case studies, features) — a horizontal scroll section feels modern and interactive.

---

## 2. Typography Techniques

### The Large Display + Tight Body Contrast
Use a very large, bold display font for headlines (clamp from 3rem to 8rem) paired with a clean, smaller body font. The contrast in scale creates hierarchy.

```css
.headline {
  font-family: 'Clash Display', sans-serif;
  font-size: clamp(3rem, 8vw, 7rem);
  font-weight: 700;
  line-height: 0.95;
  letter-spacing: -0.03em;
}
.body {
  font-family: 'Satoshi', sans-serif;
  font-size: clamp(1rem, 1.2vw, 1.125rem);
  line-height: 1.7;
  color: var(--text-secondary);
}
```

### The Mixed-Weight Headline
Within a single headline, vary weights for emphasis:
```html
<h1>
  <span class="font-light">Stop losing</span>
  <span class="font-bold">40% of your revenue</span>
  <span class="font-light">to preventable churn.</span>
</h1>
```

### The Monospace Accent
Use a monospace font for labels, tags, or small metadata. Creates a "tech" feel without being heavy-handed.

```css
.tag {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
}
```

---

## 3. Framer Motion Recipes

### Staggered Reveal
Children animate in sequence as they enter the viewport.

```tsx
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: { staggerChildren: 0.1 }
  }
};

const itemVariants = {
  hidden: { opacity: 0, y: 30 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.6, ease: [0.25, 0.46, 0.45, 0.94] }
  }
};

<motion.div
  variants={containerVariants}
  initial="hidden"
  whileInView="visible"
  viewport={{ once: true, margin: "-100px" }}
>
  {items.map(item => (
    <motion.div key={item.id} variants={itemVariants}>
      {item.content}
    </motion.div>
  ))}
</motion.div>
```

### Magnetic Button
Button that subtly follows the cursor on hover.

```tsx
function MagneticButton({ children }) {
  const ref = useRef(null);
  const x = useMotionValue(0);
  const y = useMotionValue(0);

  function handleMouse(e) {
    const rect = ref.current.getBoundingClientRect();
    const centerX = rect.left + rect.width / 2;
    const centerY = rect.top + rect.height / 2;
    x.set((e.clientX - centerX) * 0.3);
    y.set((e.clientY - centerY) * 0.3);
  }

  function reset() { x.set(0); y.set(0); }

  return (
    <motion.button
      ref={ref}
      style={{ x, y }}
      onMouseMove={handleMouse}
      onMouseLeave={reset}
      transition={{ type: "spring", stiffness: 300, damping: 20 }}
    >
      {children}
    </motion.button>
  );
}
```

### Smooth Page Transitions
Using AnimatePresence for route transitions.

```tsx
<AnimatePresence mode="wait">
  <motion.div
    key={pathname}
    initial={{ opacity: 0, y: 20 }}
    animate={{ opacity: 1, y: 0 }}
    exit={{ opacity: 0, y: -20 }}
    transition={{ duration: 0.3, ease: "easeInOut" }}
  >
    {children}
  </motion.div>
</AnimatePresence>
```

### Card Hover Lift
Subtle 3D tilt on card hover.

```tsx
<motion.div
  whileHover={{
    y: -8,
    rotateX: 2,
    rotateY: -2,
    transition: { duration: 0.3 }
  }}
  style={{ transformPerspective: 1000 }}
>
  <Card />
</motion.div>
```

---

## 4. GSAP Scroll Patterns

### Text Reveal (Character by Character)
Words or characters animate in as the user scrolls.

```js
import { gsap } from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
import { SplitText } from 'gsap/SplitText'; // Club plugin — or use a free alternative

gsap.registerPlugin(ScrollTrigger);

// Split headline into characters
const split = new SplitText('.reveal-text', { type: 'chars,words' });
gsap.from(split.chars, {
  opacity: 0,
  y: 50,
  rotateX: -90,
  stagger: 0.02,
  duration: 0.8,
  ease: 'back.out(1.7)',
  scrollTrigger: {
    trigger: '.reveal-text',
    start: 'top 80%',
    end: 'top 20%',
    toggleActions: 'play none none reverse'
  }
});
```

### Parallax Layers
Different elements move at different speeds on scroll.

```js
gsap.to('.parallax-bg', {
  yPercent: -30,
  ease: 'none',
  scrollTrigger: {
    trigger: '.parallax-section',
    start: 'top bottom',
    end: 'bottom top',
    scrub: true
  }
});

gsap.to('.parallax-fg', {
  yPercent: -60,
  ease: 'none',
  scrollTrigger: {
    trigger: '.parallax-section',
    start: 'top bottom',
    end: 'bottom top',
    scrub: true
  }
});
```

### Horizontal Scroll Section
A section that scrolls horizontally as the user scrolls vertically.

```js
const sections = gsap.utils.toArray('.horizontal-panel');
gsap.to(sections, {
  xPercent: -100 * (sections.length - 1),
  ease: 'none',
  scrollTrigger: {
    trigger: '.horizontal-container',
    pin: true,
    scrub: 1,
    snap: 1 / (sections.length - 1),
    end: () => '+=' + document.querySelector('.horizontal-container').offsetWidth
  }
});
```

### Number Counter
Animated statistics that count up on scroll.

```js
gsap.from('.stat-number', {
  textContent: 0,
  duration: 2,
  ease: 'power1.inOut',
  snap: { textContent: 1 },
  scrollTrigger: {
    trigger: '.stats-section',
    start: 'top 70%'
  }
});
```

---

## 5. Component Recipes

### The Glassmorphism Card (tasteful, not overdone)
```css
.glass-card {
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 1.5rem;
  padding: 2rem;
}
```

### The Gradient Border
```css
.gradient-border {
  position: relative;
  background: var(--bg);
  border-radius: 1rem;
  padding: 2rem;
}
.gradient-border::before {
  content: '';
  position: absolute;
  inset: -1px;
  border-radius: inherit;
  background: linear-gradient(135deg, var(--accent), transparent 60%);
  z-index: -1;
}
```

### The Spotlight Effect (cursor-following glow)
```tsx
function SpotlightCard({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  return (
    <div
      onMouseMove={(e) => {
        const rect = e.currentTarget.getBoundingClientRect();
        setPosition({ x: e.clientX - rect.left, y: e.clientY - rect.top });
      }}
      style={{
        background: `radial-gradient(400px circle at ${position.x}px ${position.y}px, rgba(255,255,255,0.06), transparent 40%)`
      }}
    >
      {children}
    </div>
  );
}
```

---

## 6. Color & Theming

### Building a palette with HSL
Start from a single hue and derive everything:

```css
:root {
  /* Primary — choose one distinctive hue */
  --hue: 250;
  --primary: hsl(var(--hue), 70%, 55%);
  --primary-light: hsl(var(--hue), 70%, 70%);
  --primary-dark: hsl(var(--hue), 70%, 40%);

  /* Neutrals — same hue, very low saturation */
  --neutral-50: hsl(var(--hue), 5%, 97%);
  --neutral-100: hsl(var(--hue), 5%, 92%);
  --neutral-200: hsl(var(--hue), 5%, 82%);
  --neutral-300: hsl(var(--hue), 5%, 68%);
  --neutral-400: hsl(var(--hue), 5%, 50%);
  --neutral-500: hsl(var(--hue), 5%, 35%);
  --neutral-600: hsl(var(--hue), 5%, 23%);
  --neutral-700: hsl(var(--hue), 5%, 15%);
  --neutral-800: hsl(var(--hue), 5%, 10%);
  --neutral-900: hsl(var(--hue), 5%, 6%);

  /* Semantic */
  --success: hsl(150, 60%, 45%);
  --warning: hsl(40, 90%, 55%);
  --error: hsl(0, 70%, 55%);
}
```

### Dark mode done right
Don't just invert colors. Dark backgrounds should be warm (slight blue or purple tint), text should be off-white (not pure #fff), and primary accents should be slightly brighter/more saturated.

---

## 7. Responsive Strategies

### Fluid everything
Use `clamp()` for font sizes, padding, and gaps so they scale smoothly:
```css
.section {
  padding: clamp(3rem, 8vw, 8rem) clamp(1rem, 5vw, 4rem);
}
h1 {
  font-size: clamp(2.5rem, 6vw, 5.5rem);
}
```

### The breakpoint mental model
Think in terms of content needs, not device names:
- 375px: Single column, stacked everything, thumb-reachable CTAs
- 768px: Two-column where it helps, navigation can expand
- 1024px: Full layout kicks in, sidebar content appears
- 1440px: Max-width container, generous whitespace

### Mobile navigation
Don't use a boring hamburger slide-in. Consider: full-screen overlay with staggered text reveal, bottom sheet navigation, or a floating action button that expands.
