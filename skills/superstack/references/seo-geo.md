# SEO & GEO Reference — Superstack

Comprehensive SEO and Generative Engine Optimization guide. Read this during Phase 10 or whenever implementing content strategy, meta tags, or structured data.

## Table of Contents
1. Technical SEO
2. Content SEO
3. Schema Markup (JSON-LD)
4. Open Graph & Social
5. Image SEO
6. GEO — Generative Engine Optimization
7. Implementation Checklist

---

## 1. Technical SEO

- **XML Sitemap**: Auto-generated, submitted to Google Search Console. Include all public pages, exclude admin/auth routes.
- **robots.txt**: Allow all public paths. Disallow `/admin`, `/api`, `/auth`.
- **Canonical URLs**: Every page gets a `<link rel="canonical">` to prevent duplicate content.
- **hreflang tags**: Required if i18n is enabled. One per language/region variant.
- **301 redirects**: For any renamed or moved URLs. Configure in `next.config.ts` or middleware.
- **Clean URL structure**: Descriptive slugs (`/pricing` not `/page-3`), no trailing slashes (pick one convention).
- **Crawlability**: No orphan pages, proper internal linking, breadcrumb navigation.

---

## 2. Content SEO

- **Heading hierarchy**: Single h1 per page, logical h2/h3 nesting
- **Meta titles**: < 60 characters, include primary keyword, unique per page
- **Meta descriptions**: < 160 characters, include CTA, unique per page
- **Internal linking**: Every page reachable within 3 clicks from homepage
- **Content freshness**: "Last updated" timestamps on content pages

---

## 3. Schema Markup (JSON-LD)

Apply the relevant schemas based on project type:

```tsx
// Organization (every site)
<script type="application/ld+json">{JSON.stringify({
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Company Name",
  "url": "https://example.com",
  "logo": "https://example.com/logo.png",
  "sameAs": ["https://twitter.com/...", "https://linkedin.com/..."]
})}</script>

// Product (SaaS)
// FAQ (landing pages)
// Article (blog posts)
// Breadcrumb (all pages with breadcrumbs)
```

---

## 4. Open Graph & Social

Every page needs full OG metadata:

```tsx
export const metadata = {
  openGraph: {
    title: 'Page Title',
    description: 'Page description',
    images: [{ url: '/og-image.png', width: 1200, height: 630 }],
    type: 'website',
  },
  twitter: {
    card: 'summary_large_image',
    title: 'Page Title',
    description: 'Page description',
    images: ['/og-image.png'],
  },
};
```

Test with Facebook Sharing Debugger and Twitter Card Validator.

---

## 5. Image SEO

- Descriptive file names (`dashboard-analytics-overview.png` not `image-1.png`)
- Alt text on all images (meaningful description, not keyword stuffing)
- Use `next/image` with WebP/AVIF formats, proper sizing, lazy loading
- Serve responsive sizes via `srcSet`

---

## 6. GEO — Generative Engine Optimization

Traditional SEO gets you clicked — GEO gets you quoted. With AI search (ChatGPT, Perplexity, Google AI Overviews) handling a growing share of queries, optimizing for AI-generated answers matters.

### Content structure for AI retrieval

AI engines break pages into individual passages and evaluate each one independently:

- Every section must stand on its own as a complete, quotable answer
- Start each section with a clear, direct answer — then expand with context
- Add FAQ sections with clear question-and-answer pairs
- Use clean heading hierarchy (H2/H3) to signal the topic of each passage
- Add brief TL;DR statements under key headings

### Technical accessibility for AI crawlers

- Check `robots.txt` — many sites block AI crawlers (GPTBot, ClaudeBot, PerplexityBot) without realizing it. Explicitly allow them if AI visibility is desired.
- Cloudflare's default config may block AI bots — verify this if using Cloudflare.
- Server-side render all important content. AI crawlers typically cannot execute JavaScript.

### Entity authority & brand signals

- Strengthen entity signals through consistent naming, structured data (JSON-LD), and cross-platform presence
- E-E-A-T (Experience, Expertise, Authoritativeness, Trustworthiness) is critical
- Transparent author bios, reputable citations, and regular content updates

### Content that gets cited by AI

- Original research, proprietary data, unique frameworks, and benchmark studies attract AI citations
- Statistics with sources: "Companies using X see a 34% improvement in Y (Source: [Study])"
- Maintain content freshness — add "Last updated" timestamps

### Cross-platform presence

- Reddit, LinkedIn, and YouTube are among the top cited sources by LLMs
- Earned media (reviews on G2/Capterra/Trustpilot) signals credibility more strongly than brand-owned content
- Get featured in existing high-ranking lists and comparison articles

### Implementation in the scaffold

- Ensure all content pages are server-side rendered
- Add FAQ schema markup (JSON-LD FAQPage) where applicable
- Structure content with clear, self-contained sections under descriptive headings
- Include `robots.txt` entries that explicitly allow AI crawlers
- Search for the latest GEO best practices during implementation — this field evolves rapidly

---

## 7. Implementation Checklist

- [ ] XML sitemap auto-generated and submitted
- [ ] robots.txt configured (public paths allowed, admin blocked, AI crawlers allowed)
- [ ] Canonical URLs on every page
- [ ] Meta title and description unique per page
- [ ] JSON-LD structured data for relevant types
- [ ] Open Graph and Twitter Card metadata complete
- [ ] All images optimized with descriptive alt text
- [ ] Internal linking structure verified
- [ ] Server-side rendering for all content pages
- [ ] FAQ sections with schema markup where applicable
