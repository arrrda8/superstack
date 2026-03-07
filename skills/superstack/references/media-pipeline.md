# Media Pipeline — Next.js

## 1. OG Images

### Dynamic OG with next/og (ImageResponse)

```typescript
// app/og/route.tsx
import { ImageResponse } from 'next/og';
import type { NextRequest } from 'next/server';

export const runtime = 'edge';

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const title = searchParams.get('title') ?? 'Default Title';
  const description = searchParams.get('description') ?? '';

  const interBold = await fetch(
    new URL('../../assets/fonts/Inter-Bold.ttf', import.meta.url)
  ).then((res) => res.arrayBuffer());

  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          flexDirection: 'column',
          justifyContent: 'center',
          padding: '80px',
          background: 'linear-gradient(135deg, #0f172a 0%, #1e293b 100%)',
          color: 'white',
          fontFamily: 'Inter',
        }}
      >
        <div style={{ fontSize: 64, fontWeight: 700, lineHeight: 1.2 }}>
          {title}
        </div>
        {description && (
          <div style={{ fontSize: 28, marginTop: 24, opacity: 0.8 }}>
            {description}
          </div>
        )}
        <div
          style={{
            display: 'flex',
            alignItems: 'center',
            marginTop: 'auto',
            fontSize: 24,
          }}
        >
          <img
            src="https://example.com/logo.png"
            width={40}
            height={40}
            style={{ marginRight: 12 }}
          />
          example.com
        </div>
      </div>
    ),
    {
      width: 1200,
      height: 630,
      fonts: [{ name: 'Inter', data: interBold, style: 'normal', weight: 700 }],
    }
  );
}
```

### Per-page OG metadata

```typescript
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [
        {
          url: `/og?title=${encodeURIComponent(post.title)}&description=${encodeURIComponent(post.excerpt)}`,
          width: 1200,
          height: 630,
          alt: post.title,
        },
      ],
    },
    twitter: {
      card: 'summary_large_image',
    },
  };
}
```

### OG template system

```typescript
// lib/og-templates.tsx
type OGTemplate = 'blog' | 'product' | 'landing';

const templates: Record<OGTemplate, (props: Record<string, string>) => JSX.Element> = {
  blog: ({ title, author, date }) => (
    <div style={{ display: 'flex', flexDirection: 'column', width: '100%', height: '100%', background: '#0f172a', color: '#fff', padding: 80 }}>
      <div style={{ fontSize: 20, opacity: 0.6, textTransform: 'uppercase' as const }}>Blog</div>
      <div style={{ fontSize: 56, fontWeight: 700, marginTop: 24 }}>{title}</div>
      <div style={{ display: 'flex', marginTop: 'auto', fontSize: 22 }}>
        <span>{author}</span>
        <span style={{ margin: '0 16px' }}>·</span>
        <span>{date}</span>
      </div>
    </div>
  ),
  product: ({ name, tagline, price }) => (
    <div style={{ display: 'flex', width: '100%', height: '100%', background: '#fff', color: '#0f172a', padding: 80 }}>
      <div style={{ display: 'flex', flexDirection: 'column', flex: 1 }}>
        <div style={{ fontSize: 56, fontWeight: 700 }}>{name}</div>
        <div style={{ fontSize: 28, marginTop: 16, opacity: 0.7 }}>{tagline}</div>
        <div style={{ fontSize: 48, marginTop: 'auto', fontWeight: 700 }}>{price}</div>
      </div>
    </div>
  ),
  landing: ({ headline }) => (
    <div style={{ display: 'flex', alignItems: 'center', justifyContent: 'center', width: '100%', height: '100%', background: 'linear-gradient(135deg, #6366f1, #8b5cf6)', color: '#fff', padding: 80 }}>
      <div style={{ fontSize: 64, fontWeight: 700, textAlign: 'center' as const }}>{headline}</div>
    </div>
  ),
};

export function getOGTemplate(template: OGTemplate, props: Record<string, string>) {
  return templates[template](props);
}
```

### Social preview testing

Use https://opengraph.xyz or https://socialsharepreview.com to verify. For automated checks:

```typescript
// e2e/og.spec.ts
import { test, expect } from '@playwright/test';

test('OG image endpoint returns valid image', async ({ request }) => {
  const response = await request.get('/og?title=Test+Post');
  expect(response.status()).toBe(200);
  expect(response.headers()['content-type']).toBe('image/png');
});

test('page has correct OG meta tags', async ({ page }) => {
  await page.goto('/blog/test-post');
  const ogTitle = await page.getAttribute('meta[property="og:title"]', 'content');
  const ogImage = await page.getAttribute('meta[property="og:image"]', 'content');
  expect(ogTitle).toBeTruthy();
  expect(ogImage).toContain('/og?');
});
```

---

## 2. Favicon System

### File structure

```
app/
  favicon.ico          # 32x32 .ico fallback
  icon.tsx             # dynamic SVG favicon (or icon.svg)
  apple-icon.tsx       # 180x180 Apple Touch Icon
  manifest.ts          # web manifest with icons
```

### Dynamic SVG favicon with dark mode

```typescript
// app/icon.tsx
import { ImageResponse } from 'next/og';

export const size = { width: 32, height: 32 };
export const contentType = 'image/png';

export default function Icon() {
  return new ImageResponse(
    (
      <div
        style={{
          width: 32,
          height: 32,
          borderRadius: 8,
          background: '#6366f1',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
          color: 'white',
          fontSize: 20,
          fontWeight: 700,
        }}
      >
        S
      </div>
    ),
    { ...size }
  );
}
```

### SVG favicon with dark mode support (static)

```html
<!-- In head or via metadata API -->
<link rel="icon" href="/favicon.svg" type="image/svg+xml" />
<link rel="icon" href="/favicon.ico" sizes="32x32" />
<link rel="apple-touch-icon" href="/apple-touch-icon.png" />
```

```svg
<!-- public/favicon.svg -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <style>
    rect { fill: #6366f1; }
    text { fill: #fff; }
    @media (prefers-color-scheme: dark) {
      rect { fill: #818cf8; }
    }
  </style>
  <rect width="32" height="32" rx="6"/>
  <text x="16" y="22" font-size="20" font-weight="bold" text-anchor="middle" font-family="system-ui">S</text>
</svg>
```

### Web manifest

```typescript
// app/manifest.ts
import type { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'My App',
    short_name: 'App',
    description: 'My Next.js application',
    start_url: '/',
    display: 'standalone',
    background_color: '#0f172a',
    theme_color: '#6366f1',
    icons: [
      { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
      { src: '/icon-maskable-512.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
    ],
  };
}
```

### Generator tools

- **realfavicongenerator.net** — most comprehensive, generates all variants
- **Sharp CLI** — `sharp -i logo.svg -o favicon.ico --resize 32 32`
- **favicons npm** — `npx favicons logo.svg -o public/`

---

## 3. Video Embedding

### Lazy-loaded YouTube (lite-youtube-embed)

```bash
npm install @nickvdh/lite-youtube-embed
# or use the web component: npm install @nickvdh/lite-youtube
```

```tsx
// components/youtube-embed.tsx
'use client';

import { useEffect, useRef } from 'react';

interface YouTubeEmbedProps {
  videoId: string;
  title: string;
}

export function YouTubeEmbed({ videoId, title }: YouTubeEmbedProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [isLoaded, setIsLoaded] = useState(false);

  return (
    <div
      ref={containerRef}
      className="relative aspect-video w-full cursor-pointer overflow-hidden rounded-xl bg-slate-900"
      onClick={() => setIsLoaded(true)}
    >
      {isLoaded ? (
        <iframe
          src={`https://www.youtube-nocookie.com/embed/${videoId}?autoplay=1`}
          title={title}
          allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
          allowFullScreen
          className="absolute inset-0 h-full w-full"
        />
      ) : (
        <>
          <img
            src={`https://i.ytimg.com/vi/${videoId}/hqdefault.jpg`}
            alt={title}
            loading="lazy"
            className="h-full w-full object-cover"
          />
          <div className="absolute inset-0 flex items-center justify-center">
            <div className="flex h-16 w-16 items-center justify-center rounded-full bg-red-600 text-white shadow-lg">
              <svg viewBox="0 0 24 24" className="ml-1 h-8 w-8" fill="currentColor">
                <path d="M8 5v14l11-7z" />
              </svg>
            </div>
          </div>
        </>
      )}
    </div>
  );
}
```

### Self-hosted video with poster

```tsx
// components/video-player.tsx
'use client';

import { useRef, useState } from 'react';

interface VideoPlayerProps {
  src: string;
  poster?: string;
  className?: string;
}

export function VideoPlayer({ src, poster, className }: VideoPlayerProps) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [isPlaying, setIsPlaying] = useState(false);

  return (
    <div className={`relative overflow-hidden rounded-xl ${className ?? ''}`}>
      <video
        ref={videoRef}
        poster={poster}
        preload="none"
        playsInline
        className="h-full w-full"
        onPlay={() => setIsPlaying(true)}
        onPause={() => setIsPlaying(false)}
      >
        <source src={src} type="video/mp4" />
        <source src={src.replace('.mp4', '.webm')} type="video/webm" />
      </video>
      {!isPlaying && (
        <button
          onClick={() => videoRef.current?.play()}
          className="absolute inset-0 flex items-center justify-center bg-black/30"
          aria-label="Play video"
        >
          <div className="flex h-16 w-16 items-center justify-center rounded-full bg-white/90">
            <svg viewBox="0 0 24 24" className="ml-1 h-8 w-8" fill="currentColor">
              <path d="M8 5v14l11-7z" />
            </svg>
          </div>
        </button>
      )}
    </div>
  );
}
```

---

## 4. SVG Handling

### SVGR setup (inline SVG as React components)

```bash
npm install -D @svgr/webpack
```

```typescript
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  webpack(config) {
    const fileLoaderRule = config.module.rules.find(
      (rule: any) => rule.test?.test?.('.svg')
    );
    config.module.rules.push(
      { ...fileLoaderRule, test: /\.svg$/i, resourceQuery: /url/ },
      {
        test: /\.svg$/i,
        resourceQuery: { not: [/url/] },
        issuer: fileLoaderRule.issuer,
        use: [{ loader: '@svgr/webpack', options: { svgo: true, svgoConfig: { plugins: [{ name: 'removeViewBox', active: false }] } } }],
      }
    );
    fileLoaderRule.exclude = /\.svg$/i;
    return config;
  },
};

export default nextConfig;
```

```tsx
// Usage
import Logo from '@/assets/logo.svg';
// <Logo className="h-8 w-8 text-indigo-500" />
```

### SVG sprite system

```tsx
// components/icon-sprite.tsx
export function IconSprite() {
  return (
    <svg xmlns="http://www.w3.org/2000/svg" className="hidden">
      <symbol id="icon-arrow-right" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth={2}>
        <path d="M5 12h14M12 5l7 7-7 7" />
      </symbol>
      <symbol id="icon-check" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth={2}>
        <path d="M20 6L9 17l-5-5" />
      </symbol>
    </svg>
  );
}

// Usage
export function SpriteIcon({ name, className }: { name: string; className?: string }) {
  return (
    <svg className={className} aria-hidden="true">
      <use href={`#icon-${name}`} />
    </svg>
  );
}
```

### Accessible SVGs

```tsx
// Decorative SVG — hidden from screen readers
<svg aria-hidden="true" focusable="false" className="h-6 w-6">
  <use href="#icon-decoration" />
</svg>

// Meaningful SVG — with accessible label
<svg role="img" aria-labelledby="chart-title chart-desc" className="h-64 w-full">
  <title id="chart-title">Monthly Revenue</title>
  <desc id="chart-desc">Bar chart showing revenue growth from Jan to Dec 2025</desc>
  {/* ... chart paths ... */}
</svg>
```

---

## 5. Image Optimization Pipeline

### Sharp for server-side processing

```typescript
// lib/image-processing.ts
import sharp from 'sharp';
import path from 'node:path';
import fs from 'node:fs/promises';

interface ProcessOptions {
  widths: number[];
  formats: ('webp' | 'avif' | 'png')[];
  quality?: number;
}

export async function processImage(
  inputPath: string,
  outputDir: string,
  options: ProcessOptions
) {
  const { widths, formats, quality = 80 } = options;
  const name = path.parse(inputPath).name;
  await fs.mkdir(outputDir, { recursive: true });

  const results: string[] = [];

  for (const width of widths) {
    for (const format of formats) {
      const outputPath = path.join(outputDir, `${name}-${width}.${format}`);
      await sharp(inputPath)
        .resize(width)
        .toFormat(format, { quality })
        .toFile(outputPath);
      results.push(outputPath);
    }
  }

  return results;
}

// Usage
await processImage('uploads/hero.jpg', 'public/images/hero', {
  widths: [640, 960, 1280, 1920],
  formats: ['avif', 'webp'],
  quality: 80,
});
```

### Responsive srcset component

```tsx
// components/optimized-image.tsx
interface OptimizedImageProps {
  src: string; // base name without extension
  alt: string;
  widths?: number[];
  sizes?: string;
  className?: string;
}

export function OptimizedImage({
  src,
  alt,
  widths = [640, 960, 1280, 1920],
  sizes = '(max-width: 640px) 100vw, (max-width: 1024px) 75vw, 50vw',
  className,
}: OptimizedImageProps) {
  const avifSrcSet = widths.map((w) => `${src}-${w}.avif ${w}w`).join(', ');
  const webpSrcSet = widths.map((w) => `${src}-${w}.webp ${w}w`).join(', ');

  return (
    <picture>
      <source type="image/avif" srcSet={avifSrcSet} sizes={sizes} />
      <source type="image/webp" srcSet={webpSrcSet} sizes={sizes} />
      <img
        src={`${src}-${widths[widths.length - 1]}.webp`}
        alt={alt}
        loading="lazy"
        decoding="async"
        className={className}
      />
    </picture>
  );
}
```

---

## 6. Content Images

### MDX images with next/image

```tsx
// components/mdx-image.tsx
import Image from 'next/image';

interface MDXImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  caption?: string;
}

export function MDXImage({ src, alt, width = 800, height = 450, caption }: MDXImageProps) {
  return (
    <figure className="my-8">
      <Image
        src={src}
        alt={alt}
        width={width}
        height={height}
        className="rounded-lg"
        placeholder="blur"
        blurDataURL={`/_next/image?url=${encodeURIComponent(src)}&w=16&q=10`}
      />
      {caption && (
        <figcaption className="mt-2 text-center text-sm text-slate-500">
          {caption}
        </figcaption>
      )}
    </figure>
  );
}
```

### Generate blur placeholder

```typescript
// lib/blur-placeholder.ts
import sharp from 'sharp';

export async function generateBlurDataURL(imagePath: string): Promise<string> {
  const buffer = await sharp(imagePath)
    .resize(10, 10, { fit: 'inside' })
    .blur()
    .toBuffer();
  return `data:image/png;base64,${buffer.toString('base64')}`;
}
```

### Art direction with picture element

```tsx
export function HeroBanner({ alt }: { alt: string }) {
  return (
    <picture>
      {/* Mobile: cropped portrait */}
      <source
        media="(max-width: 639px)"
        srcSet="/images/hero-mobile.avif"
        type="image/avif"
      />
      <source
        media="(max-width: 639px)"
        srcSet="/images/hero-mobile.webp"
        type="image/webp"
      />
      {/* Tablet */}
      <source
        media="(max-width: 1023px)"
        srcSet="/images/hero-tablet.avif"
        type="image/avif"
      />
      {/* Desktop: full landscape */}
      <source
        srcSet="/images/hero-desktop.avif"
        type="image/avif"
      />
      <img
        src="/images/hero-desktop.webp"
        alt={alt}
        className="h-[60vh] w-full object-cover"
        fetchPriority="high"
      />
    </picture>
  );
}
```

---

## 7. Icons

### Icon component wrapper (Phosphor / Tabler / Iconoir)

```bash
# Pick one:
npm install @phosphor-icons/react     # 9000+ icons, 6 weights
npm install @tabler/icons-react       # 5400+ icons
npm install iconoir-react             # 1600+ icons
```

```tsx
// components/ui/icon.tsx
import type { ComponentPropsWithoutRef } from 'react';
import type { Icon as PhosphorIcon } from '@phosphor-icons/react';

type IconSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

const sizeMap: Record<IconSize, number> = {
  xs: 14,
  sm: 16,
  md: 20,
  lg: 24,
  xl: 32,
};

interface IconProps extends Omit<ComponentPropsWithoutRef<'svg'>, 'children'> {
  icon: PhosphorIcon;
  size?: IconSize;
  label?: string; // accessible label
}

export function Icon({ icon: IconComponent, size = 'md', label, ...props }: IconProps) {
  return (
    <IconComponent
      size={sizeMap[size]}
      aria-hidden={!label}
      aria-label={label}
      role={label ? 'img' : undefined}
      {...props}
    />
  );
}
```

```tsx
// Usage
import { ArrowRight, Check, Warning } from '@phosphor-icons/react';
import { Icon } from '@/components/ui/icon';

<Icon icon={ArrowRight} size="md" />
<Icon icon={Warning} size="lg" label="Warning" className="text-amber-500" />
```

### Tree-shaking (automatic with named imports)

```tsx
// GOOD — only imports ArrowRight (tree-shakeable)
import { ArrowRight } from '@phosphor-icons/react';

// BAD — imports entire library
import * as Icons from '@phosphor-icons/react';
```

---

## 8. Audio

### Audio player component

```tsx
'use client';

import { useRef, useState, useEffect } from 'react';

interface AudioPlayerProps {
  src: string;
  title: string;
}

export function AudioPlayer({ src, title }: AudioPlayerProps) {
  const audioRef = useRef<HTMLAudioElement>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [progress, setProgress] = useState(0);
  const [duration, setDuration] = useState(0);

  useEffect(() => {
    const audio = audioRef.current;
    if (!audio) return;

    const updateProgress = () => setProgress(audio.currentTime);
    const setDur = () => setDuration(audio.duration);
    const onEnded = () => setIsPlaying(false);

    audio.addEventListener('timeupdate', updateProgress);
    audio.addEventListener('loadedmetadata', setDur);
    audio.addEventListener('ended', onEnded);

    return () => {
      audio.removeEventListener('timeupdate', updateProgress);
      audio.removeEventListener('loadedmetadata', setDur);
      audio.removeEventListener('ended', onEnded);
    };
  }, []);

  function togglePlay() {
    if (!audioRef.current) return;
    if (isPlaying) {
      audioRef.current.pause();
    } else {
      audioRef.current.play();
    }
    setIsPlaying(!isPlaying);
  }

  function seek(e: React.ChangeEvent<HTMLInputElement>) {
    const time = Number(e.target.value);
    if (audioRef.current) audioRef.current.currentTime = time;
    setProgress(time);
  }

  function formatTime(s: number) {
    const m = Math.floor(s / 60);
    const sec = Math.floor(s % 60);
    return `${m}:${sec.toString().padStart(2, '0')}`;
  }

  return (
    <div className="flex items-center gap-4 rounded-xl bg-slate-100 p-4 dark:bg-slate-800">
      <audio ref={audioRef} src={src} preload="metadata" />
      <button
        onClick={togglePlay}
        className="flex h-10 w-10 shrink-0 items-center justify-center rounded-full bg-indigo-600 text-white"
        aria-label={isPlaying ? 'Pause' : 'Play'}
      >
        {isPlaying ? '||' : '\u25B6'}
      </button>
      <div className="flex flex-1 flex-col gap-1">
        <span className="text-sm font-medium">{title}</span>
        <input
          type="range"
          min={0}
          max={duration || 0}
          value={progress}
          onChange={seek}
          className="w-full accent-indigo-600"
          aria-label="Seek"
        />
      </div>
      <span className="text-xs text-slate-500">
        {formatTime(progress)} / {formatTime(duration)}
      </span>
    </div>
  );
}
```

### Podcast embedding

```tsx
export function PodcastEmbed({ episodeUrl }: { episodeUrl: string }) {
  // Spotify
  if (episodeUrl.includes('spotify.com')) {
    const embedUrl = episodeUrl.replace('/episode/', '/embed/episode/');
    return (
      <iframe
        src={embedUrl}
        width="100%"
        height="232"
        allow="encrypted-media"
        loading="lazy"
        className="rounded-xl"
        title="Spotify episode"
      />
    );
  }

  // Apple Podcasts
  if (episodeUrl.includes('podcasts.apple.com')) {
    const embedUrl = episodeUrl.replace('podcasts.apple.com', 'embed.podcasts.apple.com');
    return (
      <iframe
        src={embedUrl}
        height="175"
        width="100%"
        allow="autoplay"
        loading="lazy"
        className="rounded-xl"
        title="Apple Podcasts episode"
      />
    );
  }

  return <AudioPlayer src={episodeUrl} title="Podcast Episode" />;
}
```

### Web Audio API basics

```typescript
// lib/audio-utils.ts
export function createAudioAnalyzer(audioElement: HTMLAudioElement) {
  const ctx = new AudioContext();
  const source = ctx.createMediaElementSource(audioElement);
  const analyser = ctx.createAnalyser();
  analyser.fftSize = 256;

  source.connect(analyser);
  analyser.connect(ctx.destination);

  const bufferLength = analyser.frequencyBinCount;
  const dataArray = new Uint8Array(bufferLength);

  return {
    getFrequencyData() {
      analyser.getByteFrequencyData(dataArray);
      return dataArray;
    },
    getWaveformData() {
      analyser.getByteTimeDomainData(dataArray);
      return dataArray;
    },
    bufferLength,
  };
}
```

---

## 9. File Type Detection

### Magic bytes validation

```typescript
// lib/file-validation.ts
const MAGIC_BYTES: Record<string, number[][]> = {
  'image/png': [[0x89, 0x50, 0x4e, 0x47]],
  'image/jpeg': [[0xff, 0xd8, 0xff]],
  'image/gif': [[0x47, 0x49, 0x46, 0x38]],
  'image/webp': [[0x52, 0x49, 0x46, 0x46]], // RIFF, check also bytes 8-11 for WEBP
  'image/avif': [], // ftyp box — check bytes 4-8 for 'ftyp'
  'application/pdf': [[0x25, 0x50, 0x44, 0x46]], // %PDF
  'image/svg+xml': [[0x3c, 0x73, 0x76, 0x67], [0x3c, 0x3f, 0x78, 0x6d]], // <svg or <?xm
};

export function detectFileType(buffer: ArrayBuffer): string | null {
  const bytes = new Uint8Array(buffer.slice(0, 12));

  for (const [mimeType, signatures] of Object.entries(MAGIC_BYTES)) {
    for (const sig of signatures) {
      if (sig.every((byte, i) => bytes[i] === byte)) {
        return mimeType;
      }
    }
  }

  return null;
}
```

### Secure upload validation

```typescript
// app/api/upload/route.ts
import { NextResponse, type NextRequest } from 'next/server';
import { detectFileType } from '@/lib/file-validation';

const ALLOWED_TYPES = new Set([
  'image/png',
  'image/jpeg',
  'image/webp',
  'image/avif',
  'application/pdf',
]);

const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10 MB

export async function POST(request: NextRequest) {
  const formData = await request.formData();
  const file = formData.get('file') as File | null;

  if (!file) {
    return NextResponse.json({ error: 'No file provided' }, { status: 400 });
  }

  // 1. Check size
  if (file.size > MAX_FILE_SIZE) {
    return NextResponse.json({ error: 'File too large' }, { status: 413 });
  }

  // 2. Check declared MIME type
  if (!ALLOWED_TYPES.has(file.type)) {
    return NextResponse.json({ error: 'File type not allowed' }, { status: 415 });
  }

  // 3. Validate magic bytes (don't trust Content-Type alone)
  const buffer = await file.arrayBuffer();
  const detectedType = detectFileType(buffer);

  if (!detectedType || !ALLOWED_TYPES.has(detectedType)) {
    return NextResponse.json(
      { error: 'File content does not match allowed types' },
      { status: 415 }
    );
  }

  // 4. Check extension matches detected type
  const ext = file.name.split('.').pop()?.toLowerCase();
  const validExtensions: Record<string, string[]> = {
    'image/png': ['png'],
    'image/jpeg': ['jpg', 'jpeg'],
    'image/webp': ['webp'],
    'image/avif': ['avif'],
    'application/pdf': ['pdf'],
  };

  if (!ext || !validExtensions[detectedType]?.includes(ext)) {
    return NextResponse.json(
      { error: 'File extension does not match content' },
      { status: 415 }
    );
  }

  // Proceed with upload...
  return NextResponse.json({ success: true });
}
```

---

## 10. CDN Strategy

### Vercel Image Optimization (built-in)

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.supabase.co',
        pathname: '/storage/v1/object/public/**',
      },
      {
        protocol: 'https',
        hostname: 'images.unsplash.com',
      },
    ],
    minimumCacheTTL: 60 * 60 * 24 * 30, // 30 days
  },
};
```

### Supabase Storage CDN

```typescript
// lib/storage.ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

export function getPublicUrl(bucket: string, path: string) {
  const { data } = supabase.storage.from(bucket).getPublicUrl(path);
  return data.publicUrl;
}

// With transformation (Supabase Pro plan)
export function getTransformedUrl(bucket: string, path: string, width: number, height: number) {
  const { data } = supabase.storage.from(bucket).getPublicUrl(path, {
    transform: { width, height, quality: 80, format: 'origin' },
  });
  return data.publicUrl;
}
```

### Cloudflare Images integration

```typescript
// lib/cloudflare-images.ts
const CF_ACCOUNT_ID = process.env.CLOUDFLARE_ACCOUNT_ID!;
const CF_API_TOKEN = process.env.CLOUDFLARE_IMAGES_TOKEN!;

export async function uploadToCloudflare(file: File) {
  const form = new FormData();
  form.append('file', file);

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/images/v1`,
    {
      method: 'POST',
      headers: { Authorization: `Bearer ${CF_API_TOKEN}` },
      body: form,
    }
  );

  const data = await response.json();
  return data.result.id;
}

// Serve with variants
export function cloudflareImageUrl(imageId: string, variant = 'public') {
  return `https://imagedelivery.net/${CF_ACCOUNT_ID}/${imageId}/${variant}`;
}
```

### Cache headers for static assets

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: '/images/:path*',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
        ],
      },
      {
        source: '/fonts/:path*',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
        ],
      },
      {
        source: '/videos/:path*',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=86400, stale-while-revalidate=604800' },
        ],
      },
    ];
  },
};
```

### CDN decision matrix

| Asset Type | Recommendation | Why |
|---|---|---|
| App images (next/image) | Vercel built-in | Zero config, auto AVIF/WebP, edge caching |
| User uploads | Supabase Storage | RLS integration, signed URLs, transformations (Pro) |
| High-volume public media | Cloudflare Images | Cheapest at scale, global edge, flexible variants |
| Video | Cloudflare Stream or Mux | Adaptive bitrate, analytics, global delivery |
| Static assets (fonts, icons) | Vercel / self-hosted + immutable headers | Small files, long cache lifetime |
