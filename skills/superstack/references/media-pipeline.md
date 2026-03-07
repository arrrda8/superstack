# Media Pipeline — Next.js Reference

## Table of Contents
- [1. OG Images](#1-og-images)
  - [Static OG with `next/og` ImageResponse](#static-og-with-nextog-imageresponse)
  - [Dynamic Per-Page OG](#dynamic-per-page-og)
  - [OG Template System](#og-template-system)
  - [Social Preview Testing](#social-preview-testing)
- [2. Favicon System](#2-favicon-system)
  - [File Convention (App Router)](#file-convention-app-router)
  - [Static Icons in `public/`](#static-icons-in-public)
  - [SVG Favicon with Dark Mode](#svg-favicon-with-dark-mode)
  - [Dynamic Favicon Generation](#dynamic-favicon-generation)
  - [Apple Touch Icon](#apple-touch-icon)
  - [Web Manifest](#web-manifest)
- [3. Video Embedding](#3-video-embedding)
  - [Lazy YouTube with lite-youtube-embed](#lazy-youtube-with-lite-youtube-embed)
  - [Self-Hosted Video with Poster Overlay](#self-hosted-video-with-poster-overlay)
  - [Poster Image Helpers](#poster-image-helpers)
- [4. SVG Handling](#4-svg-handling)
  - [Inline SVG Components](#inline-svg-components)
  - [SVGR Setup](#svgr-setup)
  - [SVG Sprite System](#svg-sprite-system)
  - [Animation-Ready SVGs](#animation-ready-svgs)
  - [SVG Accessibility Patterns](#svg-accessibility-patterns)
- [5. Image Optimization](#5-image-optimization)
  - [Sharp for Server-Side Processing](#sharp-for-server-side-processing)
  - [AVIF/WebP with next/image Config](#avifwebp-with-nextimage-config)
  - [Responsive srcset with next/image](#responsive-srcset-with-nextimage)
- [6. Content Images](#6-content-images)
  - [MDX with next/image](#mdx-with-nextimage)
  - [MDX Component Override](#mdx-component-override)
  - [Blur Placeholder Generation at Build Time](#blur-placeholder-generation-at-build-time)
  - [Art Direction with `<picture>`](#art-direction-with-picture)
  - [CMS Image with Responsive Variants](#cms-image-with-responsive-variants)
- [7. Icons](#7-icons)
  - [Recommended Libraries](#recommended-libraries)
  - [Setup and Tree-Shaking](#setup-and-tree-shaking)
  - [Custom Icon Wrapper](#custom-icon-wrapper)
  - [Consistent Sizing Rules](#consistent-sizing-rules)
  - [Icon Button Pattern](#icon-button-pattern)
- [8. Audio](#8-audio)
  - [Audio Player Component](#audio-player-component)
  - [Podcast Embedding](#podcast-embedding)
- [9. File Type Detection](#9-file-type-detection)
  - [Magic Bytes Validation](#magic-bytes-validation)
  - [MIME Checking Utility](#mime-checking-utility)
  - [Secure Upload Validation](#secure-upload-validation)
  - [Server-Side Upload Route](#server-side-upload-route)
- [10. CDN Strategy](#10-cdn-strategy)
  - [Vercel Image Optimization (Default)](#vercel-image-optimization-default)
  - [Cloudflare Images](#cloudflare-images)
  - [Supabase Storage CDN](#supabase-storage-cdn)
  - [CDN Selection Guide](#cdn-selection-guide)
  - [Custom Image Loader for External CDN](#custom-image-loader-for-external-cdn)
  - [Cache Headers for Self-Hosted Media](#cache-headers-for-self-hosted-media)

Complete reference for handling images, video, audio, icons, SVGs, OG images, favicons, file validation, and CDN strategy in a Next.js App Router project.

---

## 1. OG Images

### Static OG with `next/og` ImageResponse

```tsx
// app/opengraph-image.tsx
import { ImageResponse } from "next/og";

export const runtime = "edge";
export const alt = "Site title";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function Image() {
  const inter = await fetch(
    new URL("../assets/fonts/Inter-Bold.ttf", import.meta.url)
  ).then((res) => res.arrayBuffer());

  return new ImageResponse(
    (
      <div
        style={{
          display: "flex",
          flexDirection: "column",
          alignItems: "center",
          justifyContent: "center",
          width: "100%",
          height: "100%",
          background: "linear-gradient(135deg, #0f0f0f 0%, #1a1a2e 100%)",
          color: "#fff",
          fontFamily: "Inter",
        }}
      >
        <h1 style={{ fontSize: 64, margin: 0 }}>Your Site Title</h1>
        <p style={{ fontSize: 28, opacity: 0.7 }}>Tagline goes here</p>
      </div>
    ),
    {
      ...size,
      fonts: [{ name: "Inter", data: inter, style: "normal", weight: 700 }],
    }
  );
}
```

### Dynamic Per-Page OG

```tsx
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from "next/og";
import { getPost } from "@/lib/posts";

export const runtime = "edge";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";
export const alt = "Blog post cover";

export async function generateImageMetadata({
  params,
}: {
  params: { slug: string };
}) {
  const post = await getPost(params.slug);
  return [{ id: "og", alt: post.title, size, contentType }];
}

export default async function Image({
  params,
}: {
  params: { slug: string };
}) {
  const post = await getPost(params.slug);

  return new ImageResponse(
    (
      <div
        style={{
          display: "flex",
          flexDirection: "column",
          justifyContent: "flex-end",
          width: "100%",
          height: "100%",
          padding: 60,
          background: "#0a0a0a",
          color: "#fafafa",
        }}
      >
        <p style={{ fontSize: 24, color: "#888", margin: 0 }}>
          {post.category}
        </p>
        <h1 style={{ fontSize: 56, margin: "16px 0 0", lineHeight: 1.15 }}>
          {post.title}
        </h1>
        <p style={{ fontSize: 22, color: "#aaa", marginTop: 20 }}>
          {post.author} &middot; {post.date}
        </p>
      </div>
    ),
    { ...size }
  );
}
```

### OG Template System

```tsx
// lib/og-templates.tsx
type OGTemplate = "default" | "blog" | "product" | "event";

interface OGData {
  title: string;
  subtitle?: string;
  tag?: string;
  logoUrl?: string;
  bgColor?: string;
}

const templates: Record<OGTemplate, (data: OGData) => React.ReactElement> = {
  default: (data) => (
    <div
      style={{
        display: "flex",
        width: "100%",
        height: "100%",
        background: data.bgColor ?? "#0a0a0a",
        color: "#fff",
        alignItems: "center",
        justifyContent: "center",
        flexDirection: "column",
      }}
    >
      <h1 style={{ fontSize: 60 }}>{data.title}</h1>
      {data.subtitle && (
        <p style={{ fontSize: 28, opacity: 0.7 }}>{data.subtitle}</p>
      )}
    </div>
  ),

  blog: (data) => (
    <div
      style={{
        display: "flex",
        width: "100%",
        height: "100%",
        background: "#0f172a",
        color: "#f8fafc",
        padding: 60,
        flexDirection: "column",
        justifyContent: "flex-end",
      }}
    >
      {data.tag && (
        <span
          style={{
            fontSize: 20,
            color: "#38bdf8",
            textTransform: "uppercase",
            letterSpacing: 2,
          }}
        >
          {data.tag}
        </span>
      )}
      <h1 style={{ fontSize: 52, marginTop: 12, lineHeight: 1.2 }}>
        {data.title}
      </h1>
    </div>
  ),

  product: (data) => (
    <div
      style={{
        display: "flex",
        width: "100%",
        height: "100%",
        background: "linear-gradient(135deg, #1e1b4b, #312e81)",
        color: "#fff",
        alignItems: "center",
        justifyContent: "center",
        flexDirection: "column",
      }}
    >
      <h1 style={{ fontSize: 56 }}>{data.title}</h1>
      {data.subtitle && (
        <p style={{ fontSize: 24, color: "#c7d2fe" }}>{data.subtitle}</p>
      )}
    </div>
  ),

  event: (data) => (
    <div
      style={{
        display: "flex",
        width: "100%",
        height: "100%",
        background: "#000",
        color: "#fff",
        padding: 60,
        flexDirection: "column",
        justifyContent: "space-between",
      }}
    >
      <span style={{ fontSize: 20, color: "#f97316" }}>{data.tag}</span>
      <h1 style={{ fontSize: 64 }}>{data.title}</h1>
    </div>
  ),
};

export function renderOGTemplate(template: OGTemplate, data: OGData) {
  return templates[template](data);
}
```

### Social Preview Testing

Use [opengraph.xyz](https://www.opengraph.xyz) or [socialsharepreview.com](https://socialsharepreview.com) to verify rendering.

In development, visit `http://localhost:3000/opengraph-image` directly to preview the root OG image.

```tsx
// app/api/og-preview/route.tsx — dev-only preview endpoint
import { ImageResponse } from "next/og";
import { renderOGTemplate } from "@/lib/og-templates";

export const runtime = "edge";

export async function GET(request: Request) {
  if (process.env.NODE_ENV !== "development") {
    return new Response("Not found", { status: 404 });
  }

  const { searchParams } = new URL(request.url);
  const template = (searchParams.get("template") ?? "default") as any;
  const title = searchParams.get("title") ?? "Preview Title";

  return new ImageResponse(renderOGTemplate(template, { title }), {
    width: 1200,
    height: 630,
  });
}
// Preview: http://localhost:3000/api/og-preview?template=blog&title=Hello+World
```

---

## 2. Favicon System

### File Convention (App Router)

Place these directly in `app/`:

```
app/
  favicon.ico          # 32x32 classic favicon (auto-served at /favicon.ico)
  icon.tsx             # dynamic generated icon (or icon.png / icon.svg)
  apple-icon.tsx       # 180x180 apple touch icon (or apple-icon.png)
```

### Static Icons in `public/`

```
public/
  favicon.ico          # 32x32 (or multi-size .ico containing 16, 32, 48)
  apple-touch-icon.png # 180x180
  icon-192.png         # PWA manifest
  icon-512.png         # PWA manifest
```

### SVG Favicon with Dark Mode

```xml
<!-- app/icon.svg — place as static file, auto-detected by Next.js -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <style>
    rect { fill: #0a0a0a; }
    path { fill: #fafafa; }
    @media (prefers-color-scheme: light) {
      rect { fill: #fafafa; }
      path { fill: #0a0a0a; }
    }
  </style>
  <rect width="32" height="32" rx="6" />
  <path d="M8 24V8h6l6 10V8h4v16h-6L12 14v10H8z" />
</svg>
```

### Dynamic Favicon Generation

```tsx
// app/icon.tsx
import { ImageResponse } from "next/og";

export const size = { width: 32, height: 32 };
export const contentType = "image/png";

export default function Icon() {
  return new ImageResponse(
    (
      <div
        style={{
          width: 32,
          height: 32,
          borderRadius: 6,
          background: "#0a0a0a",
          display: "flex",
          alignItems: "center",
          justifyContent: "center",
          color: "#fafafa",
          fontSize: 20,
          fontWeight: 700,
        }}
      >
        N
      </div>
    ),
    { ...size }
  );
}
```

### Apple Touch Icon

```tsx
// app/apple-icon.tsx
import { ImageResponse } from "next/og";

export const size = { width: 180, height: 180 };
export const contentType = "image/png";

export default function AppleIcon() {
  return new ImageResponse(
    (
      <div
        style={{
          width: 180,
          height: 180,
          borderRadius: 36,
          background: "linear-gradient(135deg, #0a0a0a, #1a1a2e)",
          display: "flex",
          alignItems: "center",
          justifyContent: "center",
          color: "#fafafa",
          fontSize: 90,
          fontWeight: 800,
        }}
      >
        N
      </div>
    ),
    { ...size }
  );
}
```

### Web Manifest

```ts
// app/manifest.ts
import type { MetadataRoute } from "next";

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: "App Name",
    short_name: "App",
    start_url: "/",
    display: "standalone",
    background_color: "#0a0a0a",
    theme_color: "#0a0a0a",
    icons: [
      { src: "/icon-192.png", sizes: "192x192", type: "image/png" },
      { src: "/icon-512.png", sizes: "512x512", type: "image/png" },
      {
        src: "/icon-512.png",
        sizes: "512x512",
        type: "image/png",
        purpose: "maskable",
      },
    ],
  };
}
```

---

## 3. Video Embedding

### Lazy YouTube with lite-youtube-embed

```bash
bun add lite-youtube-embed
```

```tsx
// components/youtube-embed.tsx
"use client";

import { useEffect, useRef } from "react";

interface YouTubeEmbedProps {
  videoId: string;
  title: string;
  poster?: "default" | "mqdefault" | "hqdefault" | "maxresdefault";
}

export function YouTubeEmbed({
  videoId,
  title,
  poster = "hqdefault",
}: YouTubeEmbedProps) {
  const initialized = useRef(false);

  useEffect(() => {
    if (initialized.current) return;
    initialized.current = true;
    import("lite-youtube-embed/src/lite-yt-embed.js");
  }, []);

  return (
    <>
      <link
        rel="stylesheet"
        href="https://cdn.jsdelivr.net/npm/lite-youtube-embed@0.3.3/src/lite-yt-embed.min.css"
      />
      {/* @ts-expect-error — custom element from lite-youtube-embed */}
      <lite-youtube
        videoid={videoId}
        playlabel={`Play: ${title}`}
        posterquality={poster}
        style={{ maxWidth: "100%", borderRadius: 12 }}
      />
    </>
  );
}
```

### Self-Hosted Video with Poster Overlay

```tsx
// components/video-player.tsx
"use client";

import { useRef, useState, useCallback } from "react";
import Image from "next/image";

interface VideoPlayerProps {
  src: string;
  poster: string;
  alt: string;
  width: number;
  height: number;
}

export function VideoPlayer({
  src,
  poster,
  alt,
  width,
  height,
}: VideoPlayerProps) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [playing, setPlaying] = useState(false);

  const handlePlay = useCallback(() => {
    setPlaying(true);
    videoRef.current?.play();
  }, []);

  return (
    <div
      className="relative overflow-hidden rounded-xl"
      style={{ aspectRatio: `${width}/${height}` }}
    >
      {!playing && (
        <button
          onClick={handlePlay}
          className="absolute inset-0 z-10 flex items-center justify-center bg-black/30 transition hover:bg-black/40"
          aria-label={`Play video: ${alt}`}
        >
          <svg
            className="h-16 w-16 text-white drop-shadow-lg"
            viewBox="0 0 24 24"
            fill="currentColor"
          >
            <path d="M8 5v14l11-7z" />
          </svg>
        </button>
      )}
      {!playing && (
        <Image
          src={poster}
          alt={alt}
          fill
          className="object-cover"
          priority
        />
      )}
      <video
        ref={videoRef}
        src={src}
        controls={playing}
        playsInline
        preload="none"
        className="h-full w-full"
        width={width}
        height={height}
      />
    </div>
  );
}
```

### Poster Image Helpers

```ts
// lib/video-poster.ts

/** Generate poster at build time with ffmpeg (script, not runtime):
 *  ffmpeg -i video.mp4 -vframes 1 -q:v 2 -vf "scale=1280:-1" poster.jpg
 */

export function getYouTubePoster(
  videoId: string,
  quality:
    | "default"
    | "mqdefault"
    | "hqdefault"
    | "sddefault"
    | "maxresdefault" = "hqdefault"
): string {
  return `https://img.youtube.com/vi/${videoId}/${quality}.jpg`;
}

export function getVimeoPosterUrl(videoId: string): string {
  return `https://vumbnail.com/${videoId}.jpg`;
}
```

---

## 4. SVG Handling

### Inline SVG Components

```tsx
// components/icons/logo.tsx
interface LogoProps {
  className?: string;
  "aria-label"?: string;
}

export function Logo({ className, "aria-label": ariaLabel }: LogoProps) {
  return (
    <svg
      xmlns="http://www.w3.org/2000/svg"
      viewBox="0 0 120 32"
      className={className}
      role="img"
      aria-label={ariaLabel ?? "Company logo"}
    >
      <title>{ariaLabel ?? "Company logo"}</title>
      <path fill="currentColor" d="M10 4h12v4H14v6h6v4h-6v10h-4V4z" />
    </svg>
  );
}
```

### SVGR Setup

```bash
bun add -d @svgr/webpack
```

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  webpack(config) {
    const fileLoaderRule = config.module.rules.find(
      (rule: any) => rule.test?.test?.(".svg")
    );

    config.module.rules.push(
      // Reapply existing rule but only for *.svg?url imports
      { ...fileLoaderRule, test: /\.svg$/i, resourceQuery: /url/ },
      // Convert all other *.svg imports to React components
      {
        test: /\.svg$/i,
        issuer: fileLoaderRule.issuer,
        resourceQuery: {
          not: [...(fileLoaderRule.resourceQuery?.not || []), /url/],
        },
        use: [
          {
            loader: "@svgr/webpack",
            options: {
              svgoConfig: {
                plugins: [
                  { name: "removeViewBox", active: false },
                  { name: "removeDimensions", active: true },
                ],
              },
              svgProps: { role: "img" },
            },
          },
        ],
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
import Logo from "@/assets/logo.svg"; // as React component
import logoUrl from "@/assets/logo.svg?url"; // as URL string

export function Header() {
  return (
    <nav>
      <Logo className="h-8 w-auto text-foreground" aria-label="Home" />
    </nav>
  );
}
```

### SVG Sprite System

Create a single sprite file at `public/icons.svg`:

```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <defs>
    <symbol id="icon-arrow" viewBox="0 0 24 24">
      <path d="M5 12h14M12 5l7 7-7 7" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" />
    </symbol>
    <symbol id="icon-check" viewBox="0 0 24 24">
      <path d="M5 13l4 4L19 7" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" />
    </symbol>
  </defs>
</svg>
```

```tsx
// components/sprite-icon.tsx
interface SpriteIconProps {
  name: string;
  size?: number;
  className?: string;
  label?: string;
}

export function SpriteIcon({
  name,
  size = 24,
  className,
  label,
}: SpriteIconProps) {
  return (
    <svg
      width={size}
      height={size}
      className={className}
      role={label ? "img" : "presentation"}
      aria-label={label}
      aria-hidden={!label}
    >
      <use href={`/icons.svg#icon-${name}`} />
    </svg>
  );
}
```

### Animation-Ready SVGs

```tsx
// components/animated-checkmark.tsx
"use client";

import { motion } from "framer-motion";

export function AnimatedCheckmark({ className }: { className?: string }) {
  return (
    <svg
      viewBox="0 0 24 24"
      className={className}
      role="img"
      aria-label="Success"
    >
      <motion.circle
        cx="12"
        cy="12"
        r="10"
        stroke="currentColor"
        strokeWidth="2"
        fill="none"
        initial={{ pathLength: 0 }}
        animate={{ pathLength: 1 }}
        transition={{ duration: 0.4, ease: "easeOut" }}
      />
      <motion.path
        d="M7 13l3 3 7-7"
        stroke="currentColor"
        strokeWidth="2"
        fill="none"
        strokeLinecap="round"
        strokeLinejoin="round"
        initial={{ pathLength: 0 }}
        animate={{ pathLength: 1 }}
        transition={{ duration: 0.3, delay: 0.4, ease: "easeOut" }}
      />
    </svg>
  );
}
```

### SVG Accessibility Patterns

```tsx
// Decorative SVG — hide from screen readers
<svg aria-hidden="true" focusable="false">...</svg>

// Informative SVG — provide a label and title
<svg role="img" aria-label="Downloads chart showing 50% growth">
  <title>Downloads chart showing 50% growth</title>
  ...
</svg>

// Interactive SVG — wrap in a labeled button
<button aria-label="Close dialog">
  <svg aria-hidden="true" focusable="false">
    <path d="M6 6l12 12M18 6L6 18" />
  </svg>
</button>
```

---

## 5. Image Optimization

### Sharp for Server-Side Processing

```bash
bun add sharp
```

```ts
// lib/image-processing.ts
import sharp from "sharp";

export async function optimizeImage(
  input: Buffer,
  options: {
    width?: number;
    height?: number;
    format?: "avif" | "webp" | "jpeg";
    quality?: number;
  } = {}
): Promise<Buffer> {
  const { width, height, format = "webp", quality = 80 } = options;

  let pipeline = sharp(input);

  if (width || height) {
    pipeline = pipeline.resize(width, height, {
      fit: "cover",
      withoutEnlargement: true,
    });
  }

  switch (format) {
    case "avif":
      pipeline = pipeline.avif({ quality, effort: 4 });
      break;
    case "webp":
      pipeline = pipeline.webp({ quality, effort: 4 });
      break;
    case "jpeg":
      pipeline = pipeline.jpeg({ quality, mozjpeg: true });
      break;
  }

  return pipeline.toBuffer();
}

export async function generateResponsiveSet(
  input: Buffer,
  widths: number[] = [640, 750, 828, 1080, 1200, 1920]
): Promise<Map<number, Buffer>> {
  const results = new Map<number, Buffer>();

  await Promise.all(
    widths.map(async (w) => {
      const buf = await optimizeImage(input, { width: w, format: "webp" });
      results.set(w, buf);
    })
  );

  return results;
}

export async function getImageMetadata(input: Buffer) {
  const metadata = await sharp(input).metadata();
  return {
    width: metadata.width ?? 0,
    height: metadata.height ?? 0,
    format: metadata.format,
    size: metadata.size,
    hasAlpha: metadata.hasAlpha ?? false,
  };
}
```

### AVIF/WebP with next/image Config

```ts
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    formats: ["image/avif", "image/webp"],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    minimumCacheTTL: 60 * 60 * 24 * 30, // 30 days
  },
};
```

### Responsive srcset with next/image

```tsx
// components/responsive-image.tsx
import Image from "next/image";

interface ResponsiveImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  sizes?: string;
  priority?: boolean;
  className?: string;
}

export function ResponsiveImage({
  src,
  alt,
  width,
  height,
  sizes = "(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw",
  priority = false,
  className,
}: ResponsiveImageProps) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      sizes={sizes}
      priority={priority}
      quality={80}
      className={className}
      placeholder="blur"
      blurDataURL={`data:image/svg+xml;base64,${btoa(
        '<svg xmlns="http://www.w3.org/2000/svg" width="40" height="40"><rect width="100%" height="100%" fill="#1a1a1a"/></svg>'
      )}`}
    />
  );
}
```

---

## 6. Content Images

### MDX with next/image

```tsx
// components/mdx/mdx-image.tsx
import Image from "next/image";
import { getPlaiceholder } from "plaiceholder";

interface MdxImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  caption?: string;
}

export async function MdxImage({
  src,
  alt,
  width = 800,
  height = 450,
  caption,
}: MdxImageProps) {
  let blurDataURL: string | undefined;

  try {
    const buffer = await fetch(src).then(async (res) =>
      Buffer.from(await res.arrayBuffer())
    );
    const { base64 } = await getPlaiceholder(buffer, { size: 10 });
    blurDataURL = base64;
  } catch {
    // Fallback: no blur placeholder
  }

  return (
    <figure className="my-8">
      <Image
        src={src}
        alt={alt}
        width={width}
        height={height}
        className="rounded-lg"
        placeholder={blurDataURL ? "blur" : "empty"}
        blurDataURL={blurDataURL}
      />
      {caption && (
        <figcaption className="mt-2 text-center text-sm text-muted-foreground">
          {caption}
        </figcaption>
      )}
    </figure>
  );
}
```

### MDX Component Override

```tsx
// lib/mdx-components.tsx
import type { MDXComponents } from "mdx/types";
import Image from "next/image";

export function useMDXComponents(components: MDXComponents): MDXComponents {
  return {
    ...components,
    img: ({ src, alt, ...props }) => {
      if (!src) return null;
      return (
        <Image
          src={src}
          alt={alt ?? ""}
          width={800}
          height={450}
          className="my-6 rounded-lg"
          sizes="(max-width: 768px) 100vw, 800px"
          {...props}
        />
      );
    },
  };
}
```

### Blur Placeholder Generation at Build Time

```ts
// lib/blur-placeholder.ts
import { getPlaiceholder } from "plaiceholder";
import fs from "node:fs/promises";
import path from "node:path";

export async function getBlurPlaceholder(
  imagePath: string
): Promise<string> {
  const isRemote = imagePath.startsWith("http");
  let buffer: Buffer;

  if (isRemote) {
    buffer = Buffer.from(await (await fetch(imagePath)).arrayBuffer());
  } else {
    const fullPath = path.join(process.cwd(), "public", imagePath);
    buffer = await fs.readFile(fullPath);
  }

  const { base64 } = await getPlaiceholder(buffer, { size: 10 });
  return base64;
}
```

### Art Direction with `<picture>`

```tsx
// components/art-directed-image.tsx
interface ArtDirectedImageProps {
  mobile: { src: string; width: number; height: number };
  tablet: { src: string; width: number; height: number };
  desktop: { src: string; width: number; height: number };
  alt: string;
}

export function ArtDirectedImage({
  mobile,
  tablet,
  desktop,
  alt,
}: ArtDirectedImageProps) {
  return (
    <picture>
      <source
        media="(min-width: 1024px)"
        srcSet={desktop.src}
        width={desktop.width}
        height={desktop.height}
      />
      <source
        media="(min-width: 640px)"
        srcSet={tablet.src}
        width={tablet.width}
        height={tablet.height}
      />
      {/* eslint-disable-next-line @next/next/no-img-element */}
      <img
        src={mobile.src}
        alt={alt}
        width={mobile.width}
        height={mobile.height}
        loading="lazy"
        decoding="async"
        className="h-auto w-full rounded-lg"
      />
    </picture>
  );
}
```

### CMS Image with Responsive Variants

```tsx
// components/cms-image.tsx
import Image from "next/image";
import { getBlurPlaceholder } from "@/lib/blur-placeholder";

interface CMSImageProps {
  src: string;
  alt: string;
  aspectRatio?: "16/9" | "4/3" | "1/1" | "3/2";
  priority?: boolean;
}

const aspectDimensions = {
  "16/9": { width: 1200, height: 675 },
  "4/3": { width: 1200, height: 900 },
  "1/1": { width: 1200, height: 1200 },
  "3/2": { width: 1200, height: 800 },
};

export async function CMSImage({
  src,
  alt,
  aspectRatio = "16/9",
  priority = false,
}: CMSImageProps) {
  const { width, height } = aspectDimensions[aspectRatio];
  const blurDataURL = await getBlurPlaceholder(src);

  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 75vw, 50vw"
      priority={priority}
      placeholder="blur"
      blurDataURL={blurDataURL}
      className="rounded-lg object-cover"
      style={{ aspectRatio }}
    />
  );
}
```

---

## 7. Icons

### Recommended Libraries

| Library | Style | Bundle | Install |
|---------|-------|--------|---------|
| **Phosphor** | Versatile, 6 weights | Tree-shakeable | `@phosphor-icons/react` |
| **Tabler** | Clean line icons | Tree-shakeable | `@tabler/icons-react` |
| **Iconoir** | Minimal, consistent | Tree-shakeable | `iconoir-react` |

### Setup and Tree-Shaking

```bash
bun add @phosphor-icons/react
```

```tsx
// Always import individual icons — never the barrel export
import { ArrowRight, Check, X } from "@phosphor-icons/react";

// NEVER do this — kills tree-shaking and bloats the bundle:
// import * as Icons from "@phosphor-icons/react";
```

### Custom Icon Wrapper

```tsx
// components/ui/icon.tsx
import type { ComponentProps, ElementType } from "react";
import { cn } from "@/lib/utils";

type IconSize = "xs" | "sm" | "md" | "lg" | "xl";

const sizeMap: Record<IconSize, number> = {
  xs: 14,
  sm: 16,
  md: 20,
  lg: 24,
  xl: 32,
};

interface IconProps {
  icon: ElementType;
  size?: IconSize;
  className?: string;
  label?: string;
}

export function Icon({
  icon: IconComponent,
  size = "md",
  className,
  label,
  ...props
}: IconProps & Omit<ComponentProps<"svg">, "ref">) {
  const px = sizeMap[size];

  return (
    <IconComponent
      size={px}
      className={cn("shrink-0", className)}
      aria-hidden={!label}
      aria-label={label}
      role={label ? "img" : "presentation"}
      {...props}
    />
  );
}
```

```tsx
// Usage
import { ArrowRight, Check } from "@phosphor-icons/react";
import { Icon } from "@/components/ui/icon";

<Icon icon={ArrowRight} size="sm" />
<Icon icon={Check} size="md" label="Completed" className="text-green-500" />
```

### Consistent Sizing Rules

```
xs (14px) — inline with small text, badges
sm (16px) — inline with body text, table cells
md (20px) — buttons, nav items, form fields (default)
lg (24px) — section headers, feature cards
xl (32px) — hero sections, empty states
```

### Icon Button Pattern

```tsx
// components/ui/icon-button.tsx
import { cn } from "@/lib/utils";
import type { ElementType, ButtonHTMLAttributes } from "react";
import { Icon, type IconSize } from "./icon";

interface IconButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  icon: ElementType;
  label: string;
  size?: IconSize;
  variant?: "ghost" | "outline" | "solid";
}

export function IconButton({
  icon,
  label,
  size = "md",
  variant = "ghost",
  className,
  ...props
}: IconButtonProps) {
  return (
    <button
      aria-label={label}
      className={cn(
        "inline-flex items-center justify-center rounded-md transition-colors",
        "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring",
        {
          ghost: "hover:bg-accent hover:text-accent-foreground",
          outline: "border hover:bg-accent",
          solid: "bg-primary text-primary-foreground hover:bg-primary/90",
        }[variant],
        {
          xs: "h-6 w-6",
          sm: "h-7 w-7",
          md: "h-9 w-9",
          lg: "h-10 w-10",
          xl: "h-12 w-12",
        }[size],
        className
      )}
      {...props}
    >
      <Icon icon={icon} size={size} />
    </button>
  );
}
```

---

## 8. Audio

### Audio Player Component

```tsx
// components/audio-player.tsx
"use client";

import { useRef, useState, useEffect, useCallback } from "react";

interface AudioPlayerProps {
  src: string;
  title: string;
  subtitle?: string;
}

export function AudioPlayer({ src, title, subtitle }: AudioPlayerProps) {
  const audioRef = useRef<HTMLAudioElement>(null);
  const [playing, setPlaying] = useState(false);
  const [currentTime, setCurrentTime] = useState(0);
  const [duration, setDuration] = useState(0);
  const [playbackRate, setPlaybackRate] = useState(1);

  useEffect(() => {
    const audio = audioRef.current;
    if (!audio) return;

    const onLoaded = () => setDuration(audio.duration);
    const onTimeUpdate = () => setCurrentTime(audio.currentTime);
    const onEnded = () => setPlaying(false);

    audio.addEventListener("loadedmetadata", onLoaded);
    audio.addEventListener("timeupdate", onTimeUpdate);
    audio.addEventListener("ended", onEnded);

    return () => {
      audio.removeEventListener("loadedmetadata", onLoaded);
      audio.removeEventListener("timeupdate", onTimeUpdate);
      audio.removeEventListener("ended", onEnded);
    };
  }, []);

  const togglePlay = useCallback(() => {
    const audio = audioRef.current;
    if (!audio) return;
    if (playing) {
      audio.pause();
    } else {
      audio.play();
    }
    setPlaying((p) => !p);
  }, [playing]);

  const seek = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const audio = audioRef.current;
    if (!audio) return;
    const time = Number(e.target.value);
    audio.currentTime = time;
    setCurrentTime(time);
  }, []);

  const cycleRate = useCallback(() => {
    const rates = [1, 1.25, 1.5, 1.75, 2];
    const next = rates[(rates.indexOf(playbackRate) + 1) % rates.length];
    if (audioRef.current) audioRef.current.playbackRate = next;
    setPlaybackRate(next);
  }, [playbackRate]);

  const skip = useCallback(
    (seconds: number) => {
      const audio = audioRef.current;
      if (!audio) return;
      audio.currentTime = Math.max(
        0,
        Math.min(audio.currentTime + seconds, duration)
      );
    },
    [duration]
  );

  function formatTime(s: number): string {
    if (!isFinite(s)) return "0:00";
    const m = Math.floor(s / 60);
    const sec = Math.floor(s % 60);
    return `${m}:${sec.toString().padStart(2, "0")}`;
  }

  return (
    <div className="rounded-xl border bg-card p-4">
      <audio ref={audioRef} src={src} preload="metadata" />
      <div className="mb-3">
        <p className="text-sm font-medium">{title}</p>
        {subtitle && (
          <p className="text-xs text-muted-foreground">{subtitle}</p>
        )}
      </div>
      <div className="flex items-center gap-3">
        <button
          onClick={() => skip(-15)}
          className="text-xs text-muted-foreground hover:text-foreground"
          aria-label="Rewind 15 seconds"
        >
          -15s
        </button>
        <button
          onClick={togglePlay}
          className="flex h-10 w-10 items-center justify-center rounded-full bg-primary text-primary-foreground"
          aria-label={playing ? "Pause" : "Play"}
        >
          {playing ? "\u23F8" : "\u25B6"}
        </button>
        <button
          onClick={() => skip(30)}
          className="text-xs text-muted-foreground hover:text-foreground"
          aria-label="Forward 30 seconds"
        >
          +30s
        </button>
        <input
          type="range"
          min={0}
          max={duration || 0}
          value={currentTime}
          onChange={seek}
          className="flex-1"
          aria-label="Seek"
        />
        <span className="text-xs tabular-nums text-muted-foreground">
          {formatTime(currentTime)} / {formatTime(duration)}
        </span>
        <button
          onClick={cycleRate}
          className="rounded border px-2 py-0.5 text-xs"
          aria-label={`Playback speed ${playbackRate}x`}
        >
          {playbackRate}x
        </button>
      </div>
    </div>
  );
}
```

### Podcast Embedding

```tsx
// components/podcast-embed.tsx
interface PodcastEmbedProps {
  platform: "spotify" | "apple";
  showId: string;
  episodeId?: string;
  theme?: "light" | "dark";
}

export function PodcastEmbed({
  platform,
  showId,
  episodeId,
  theme = "dark",
}: PodcastEmbedProps) {
  if (platform === "spotify") {
    const type = episodeId ? "episode" : "show";
    const id = episodeId ?? showId;
    return (
      <iframe
        src={`https://open.spotify.com/embed/${type}/${id}?theme=${theme === "dark" ? 0 : 1}`}
        width="100%"
        height={episodeId ? 152 : 352}
        allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture"
        loading="lazy"
        className="rounded-xl"
        title="Spotify podcast player"
      />
    );
  }

  // Apple Podcasts
  const path = episodeId ? `${showId}?i=${episodeId}` : showId;
  return (
    <iframe
      src={`https://embed.podcasts.apple.com/podcast/id${path}&theme=${theme}`}
      width="100%"
      height={episodeId ? 175 : 450}
      allow="autoplay"
      loading="lazy"
      className="rounded-xl"
      title="Apple Podcasts player"
      sandbox="allow-forms allow-popups allow-same-origin allow-scripts allow-top-navigation-by-user-activation"
    />
  );
}
```

---

## 9. File Type Detection

### Magic Bytes Validation

```ts
// lib/file-validation.ts

const MAGIC_BYTES: Record<string, { bytes: number[]; offset?: number }[]> = {
  "image/jpeg": [{ bytes: [0xff, 0xd8, 0xff] }],
  "image/png": [
    { bytes: [0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a] },
  ],
  "image/gif": [{ bytes: [0x47, 0x49, 0x46, 0x38] }],
  "image/webp": [{ bytes: [0x52, 0x49, 0x46, 0x46], offset: 0 }],
  "image/avif": [{ bytes: [0x00, 0x00, 0x00], offset: 0 }],
  "image/svg+xml": [{ bytes: [0x3c, 0x3f, 0x78, 0x6d, 0x6c] }], // <?xml
  "application/pdf": [{ bytes: [0x25, 0x50, 0x44, 0x46] }], // %PDF
  "audio/mpeg": [{ bytes: [0x49, 0x44, 0x33] }], // ID3
  "video/mp4": [{ bytes: [0x66, 0x74, 0x79, 0x70], offset: 4 }], // ftyp
};

export function detectMimeType(buffer: ArrayBuffer): string | null {
  const view = new Uint8Array(buffer);

  for (const [mime, signatures] of Object.entries(MAGIC_BYTES)) {
    for (const sig of signatures) {
      const offset = sig.offset ?? 0;
      const match = sig.bytes.every(
        (byte, i) => view[offset + i] === byte
      );
      if (match) return mime;
    }
  }

  return null;
}

export function isSVG(buffer: ArrayBuffer): boolean {
  const text = new TextDecoder().decode(buffer.slice(0, 500));
  return text.includes("<svg") || text.startsWith("<?xml");
}
```

### MIME Checking Utility

```ts
// lib/mime.ts

const EXTENSION_TO_MIME: Record<string, string> = {
  jpg: "image/jpeg",
  jpeg: "image/jpeg",
  png: "image/png",
  gif: "image/gif",
  webp: "image/webp",
  avif: "image/avif",
  svg: "image/svg+xml",
  mp4: "video/mp4",
  webm: "video/webm",
  mp3: "audio/mpeg",
  wav: "audio/wav",
  ogg: "audio/ogg",
  pdf: "application/pdf",
  woff2: "font/woff2",
  woff: "font/woff",
};

export function mimeFromExtension(filename: string): string | null {
  const ext = filename.split(".").pop()?.toLowerCase();
  return ext ? EXTENSION_TO_MIME[ext] ?? null : null;
}

export function extensionFromMime(mime: string): string | null {
  for (const [ext, m] of Object.entries(EXTENSION_TO_MIME)) {
    if (m === mime) return ext;
  }
  return null;
}
```

### Secure Upload Validation

```ts
// lib/upload-validator.ts
import { detectMimeType, isSVG } from "./file-validation";
import { mimeFromExtension } from "./mime";

interface ValidationResult {
  valid: boolean;
  mime: string | null;
  error?: string;
}

const ALLOWED_IMAGE_TYPES = new Set([
  "image/jpeg",
  "image/png",
  "image/webp",
  "image/avif",
  "image/gif",
]);

const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10 MB

export async function validateUpload(
  file: File,
  options: {
    allowedTypes?: Set<string>;
    maxSize?: number;
    allowSvg?: boolean;
  } = {}
): Promise<ValidationResult> {
  const {
    allowedTypes = ALLOWED_IMAGE_TYPES,
    maxSize = MAX_FILE_SIZE,
    allowSvg = false,
  } = options;

  // 1. Size check
  if (file.size > maxSize) {
    return {
      valid: false,
      mime: null,
      error: `File exceeds maximum size of ${Math.round(maxSize / 1024 / 1024)}MB`,
    };
  }

  // 2. Read header bytes for magic number detection
  const headerSlice = await file.slice(0, 16).arrayBuffer();
  const detectedMime = detectMimeType(headerSlice);

  // 3. SVG special case (text-based, needs sanitization before use)
  if (!detectedMime && allowSvg) {
    const fullBuffer = await file.arrayBuffer();
    if (isSVG(fullBuffer)) {
      return { valid: true, mime: "image/svg+xml" };
    }
  }

  // 4. Verify detected MIME is in the allowed list
  if (!detectedMime || !allowedTypes.has(detectedMime)) {
    return {
      valid: false,
      mime: detectedMime,
      error: `File type ${detectedMime ?? "unknown"} is not allowed`,
    };
  }

  // 5. Cross-check file extension vs detected magic bytes
  const extensionMime = mimeFromExtension(file.name);
  if (extensionMime && extensionMime !== detectedMime) {
    return {
      valid: false,
      mime: detectedMime,
      error: `Extension mismatch: .${file.name.split(".").pop()} does not match detected type ${detectedMime}`,
    };
  }

  return { valid: true, mime: detectedMime };
}
```

### Server-Side Upload Route

```ts
// app/api/upload/route.ts
import { NextResponse } from "next/server";
import { validateUpload } from "@/lib/upload-validator";

export async function POST(request: Request) {
  const formData = await request.formData();
  const file = formData.get("file") as File | null;

  if (!file) {
    return NextResponse.json({ error: "No file provided" }, { status: 400 });
  }

  const validation = await validateUpload(file);

  if (!validation.valid) {
    return NextResponse.json({ error: validation.error }, { status: 400 });
  }

  // Proceed with upload to Supabase Storage / S3 / etc.
  const buffer = Buffer.from(await file.arrayBuffer());
  // ... upload logic here

  return NextResponse.json({ success: true, mime: validation.mime });
}
```

---

## 10. CDN Strategy

### Vercel Image Optimization (Default)

```ts
// next.config.ts — automatic when deploying on Vercel
const nextConfig: NextConfig = {
  images: {
    formats: ["image/avif", "image/webp"],
    remotePatterns: [
      { protocol: "https", hostname: "**.supabase.co" },
      { protocol: "https", hostname: "images.unsplash.com" },
    ],
    minimumCacheTTL: 2592000, // 30 days
  },
};
```

Usage is automatic via `next/image`:

```tsx
import Image from "next/image";

<Image
  src="https://yourproject.supabase.co/storage/v1/object/public/images/hero.jpg"
  alt="Hero"
  width={1200}
  height={630}
/>
```

### Cloudflare Images

```ts
// lib/cdn/cloudflare.ts
const CF_ACCOUNT_HASH = process.env.CLOUDFLARE_ACCOUNT_HASH!;

type CfVariant = "public" | "thumbnail" | "avatar" | "og";

export function cfImageUrl(
  imageId: string,
  variant: CfVariant = "public"
): string {
  return `https://imagedelivery.net/${CF_ACCOUNT_HASH}/${imageId}/${variant}`;
}

// Flexible transforms (requires Cloudflare Images with transformations enabled)
export function cfTransformUrl(
  originalUrl: string,
  options: {
    width?: number;
    height?: number;
    fit?: "scale-down" | "contain" | "cover" | "crop";
    quality?: number;
    format?: "auto" | "avif" | "webp";
  }
): string {
  const params = Object.entries(options)
    .filter(([, v]) => v !== undefined)
    .map(([k, v]) => `${k}=${v}`)
    .join(",");

  return `/cdn-cgi/image/${params}/${originalUrl}`;
}
```

### Supabase Storage CDN

```ts
// lib/cdn/supabase-storage.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

export function getPublicUrl(bucket: string, path: string): string {
  const { data } = supabase.storage.from(bucket).getPublicUrl(path);
  return data.publicUrl;
}

// With Supabase Image Transformation (requires Pro plan)
export function getTransformedUrl(
  bucket: string,
  path: string,
  options: {
    width?: number;
    height?: number;
    quality?: number;
    format?: "origin";
  } = {}
): string {
  const { data } = supabase.storage
    .from(bucket)
    .getPublicUrl(path, { transform: options });
  return data.publicUrl;
}

// Upload with optimized cache headers
export async function uploadImage(
  bucket: string,
  path: string,
  file: File
): Promise<string> {
  const { error } = await supabase.storage.from(bucket).upload(path, file, {
    cacheControl: "public, max-age=31536000, immutable",
    contentType: file.type,
    upsert: false,
  });

  if (error) throw error;
  return getPublicUrl(bucket, path);
}
```

### CDN Selection Guide

| Use Case | Recommended | Why |
|----------|-------------|-----|
| Standard Next.js on Vercel | Vercel Image Optimization | Zero config, automatic AVIF/WebP |
| High-volume image-heavy (e-commerce) | Cloudflare Images | Per-image pricing, global PoPs, flexible variants |
| App with Supabase backend | Supabase Storage + Vercel loader | Single ecosystem, RLS on storage |
| User-uploaded content | Supabase Storage (upload) + Vercel/CF (delivery) | Secure upload with optimized delivery |
| Static marketing site | Vercel + `public/` folder | Simplest setup, CDN-cached automatically |

### Custom Image Loader for External CDN

```ts
// lib/image-loader.ts
import type { ImageLoaderProps } from "next/image";

export function supabaseLoader({
  src,
  width,
  quality,
}: ImageLoaderProps): string {
  const url = new URL(src);
  url.searchParams.set("width", width.toString());
  url.searchParams.set("quality", (quality ?? 75).toString());
  return url.toString();
}
```

```ts
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    loader: "custom",
    loaderFile: "./lib/image-loader.ts",
  },
};
```

### Cache Headers for Self-Hosted Media

```ts
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Aggressive caching for static media assets
  if (
    pathname.match(
      /\.(jpg|jpeg|png|webp|avif|gif|svg|ico|mp4|webm|mp3|woff2)$/
    )
  ) {
    const response = NextResponse.next();
    response.headers.set(
      "Cache-Control",
      "public, max-age=31536000, immutable"
    );
    return response;
  }
}

export const config = {
  matcher: ["/media/:path*", "/fonts/:path*"],
};
```
