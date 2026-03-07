# Data Visualization in React/Next.js

## Table of Contents
- [1. Library Comparison](#1-library-comparison)
- [2. Recharts Setup](#2-recharts-setup)
  - [Installation and SSR-Safe Import](#installation-and-ssr-safe-import)
  - [OKLCH Color Palette](#oklch-color-palette)
  - [ResponsiveContainer (Always Wrap Charts)](#responsivecontainer-always-wrap-charts)
  - [Line Chart](#line-chart)
  - [Bar Chart](#bar-chart)
  - [Area Chart](#area-chart)
  - [Pie / Donut Chart](#pie--donut-chart)
  - [Custom Tooltip](#custom-tooltip)
- [3. Tremor -- Dashboard Components](#3-tremor----dashboard-components)
  - [Installation](#installation)
  - [KPI Cards](#kpi-cards)
  - [Sparklines and Mini Charts](#sparklines-and-mini-charts)
  - [Tremor Bar/Line Charts (Higher-Level API)](#tremor-barline-charts-higher-level-api)
- [4. Dashboard Layouts](#4-dashboard-layouts)
  - [Grid Pattern for KPI Dashboard](#grid-pattern-for-kpi-dashboard)
  - [Reusable Chart Card Component](#reusable-chart-card-component)
  - [Responsive Breakpoints Reference](#responsive-breakpoints-reference)
- [5. Responsive Charts](#5-responsive-charts)
  - [Mobile-Friendly Chart Pattern](#mobile-friendly-chart-pattern)
  - [useMediaQuery Hook](#usemediaquery-hook)
  - [Simplified Mobile View (Table Fallback)](#simplified-mobile-view-table-fallback)
- [6. Real-Time Charts](#6-real-time-charts)
  - [Streaming Data with Supabase Realtime](#streaming-data-with-supabase-realtime)
  - [Animated Value Change](#animated-value-change)
  - [Smooth Animation on Data Change](#smooth-animation-on-data-change)
  - [Polling Fallback (When Realtime Is Not Available)](#polling-fallback-when-realtime-is-not-available)
- [7. Data Formatting](#7-data-formatting)
  - [Number Formatting Utilities](#number-formatting-utilities)
  - [Using Formatters in Charts](#using-formatters-in-charts)
- [8. Export -- PNG, CSV, PDF](#8-export----png-csv-pdf)
  - [Chart to PNG (html2canvas)](#chart-to-png-html2canvas)
  - [Data to CSV](#data-to-csv)
  - [PDF Export (jsPDF + autotable)](#pdf-export-jspdf--autotable)
  - [PDF Report Generation (react-pdf -- Alternative)](#pdf-report-generation-react-pdf----alternative)
- [9. Accessible Charts](#9-accessible-charts)
  - [ARIA Labels](#aria-labels)
  - [Color-Blind Safe Palettes](#color-blind-safe-palettes)
  - [Data Table Alternative Toggle](#data-table-alternative-toggle)
- [10. Common Chart Types -- When to Use](#10-common-chart-types----when-to-use)
  - [Funnel Chart](#funnel-chart)
  - [Composed Chart (Bar + Line Overlay)](#composed-chart-bar--line-overlay)
  - [Heatmap (Custom with CSS Grid)](#heatmap-custom-with-css-grid)
- [Quick Decision Guide](#quick-decision-guide)

Reference for building charts, dashboards, and data-driven UIs. Primary recommendation: **Recharts**.

---

## 1. Library Comparison

| Feature | Recharts | Tremor | Chart.js (react-chartjs-2) | Nivo |
|---|---|---|---|---|
| Bundle size (gzip) | ~45 KB | ~90 KB (includes Tremor UI) | ~35 KB | ~60 KB per chart type |
| SSR support | Good (SVG-based, dynamic import) | Good (built for Next.js) | Poor (canvas-based, needs `"use client"`) | Partial (SVG mode works) |
| Customization | High (composable API) | Medium (opinionated design) | High (imperative config) | Very High (theming system) |
| TypeScript | Excellent | Excellent | Good | Excellent |
| Learning curve | Low | Very Low | Medium | Medium-High |
| React integration | Native (declarative JSX) | Native (Tailwind + React) | Wrapper library | Native (declarative JSX) |
| Animation | Built-in | Built-in | Built-in | Built-in (spring physics) |
| Accessibility | Manual | Basic built-in | Poor | Good (built-in ARIA) |
| Active maintenance | Active | Active | Active | Active |

**Decision guide:**
- **Recharts** -- Best all-around for custom dashboards. Composable, flexible, good docs.
- **Tremor** -- Best for shipping fast. Pre-built dashboard components with KPI cards and sparklines. Less control.
- **Chart.js** -- Best if you need canvas rendering for large datasets (10K+ points). Poor SSR.
- **Nivo** -- Best for advanced/exotic chart types (sunburst, chord, sankey). Heavier.

**Recommendation:** Use Recharts for most projects. Add Tremor for rapid prototyping or internal tools.

```bash
# Primary setup
bun add recharts
# For dashboard components
bun add @tremor/react
```

---

## 2. Recharts Setup

### Installation and SSR-Safe Import

```bash
bun add recharts
```

```tsx
// components/charts/chart-wrapper.tsx
"use client";

import dynamic from "next/dynamic";

// Recharts uses window internally, so dynamic import with ssr: false
const Chart = dynamic(() => import("./my-chart"), { ssr: false });

export function ChartWrapper(props: ChartProps) {
  return <Chart {...props} />;
}
```

### OKLCH Color Palette

```ts
// lib/chart-colors.ts
export const CHART_COLORS = {
  primary: "oklch(0.65 0.2 250)",    // blue
  success: "oklch(0.7 0.18 150)",    // green
  warning: "oklch(0.75 0.15 80)",    // yellow
  danger: "oklch(0.65 0.22 25)",     // red
  info: "oklch(0.7 0.15 200)",       // cyan
  purple: "oklch(0.6 0.2 290)",      // purple
  pink: "oklch(0.65 0.22 330)",      // pink
  orange: "oklch(0.7 0.2 50)",       // orange
} as const;

// Ordered palette for multi-series charts
export const SERIES_COLORS = [
  CHART_COLORS.primary,
  CHART_COLORS.success,
  CHART_COLORS.warning,
  CHART_COLORS.danger,
  CHART_COLORS.purple,
  CHART_COLORS.info,
  CHART_COLORS.pink,
  CHART_COLORS.orange,
];

// Color-blind safe palette (Wong palette adapted to OKLCH)
export const COLORBLIND_PALETTE = [
  "oklch(0.45 0.18 260)",  // dark blue
  "oklch(0.75 0.18 85)",   // yellow/orange
  "oklch(0.55 0.00 0)",    // dark gray
  "oklch(0.70 0.20 30)",   // orange
  "oklch(0.60 0.15 200)",  // teal
  "oklch(0.80 0.10 140)",  // light green
];
```

### ResponsiveContainer (Always Wrap Charts)

```tsx
// IMPORTANT: ResponsiveContainer needs a parent with defined height
export function ChartFrame({
  children,
  height = 350,
}: {
  children: React.ReactNode;
  height?: number;
}) {
  return (
    <div style={{ width: "100%", height }}>
      <ResponsiveContainer width="100%" height="100%">
        {children}
      </ResponsiveContainer>
    </div>
  );
}
```

### Line Chart

```tsx
// components/charts/revenue-chart.tsx
"use client";

import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
} from "recharts";

interface DataPoint {
  date: string;
  revenue: number;
  target: number;
}

export function RevenueChart({ data }: { data: DataPoint[] }) {
  return (
    <ResponsiveContainer width="100%" height={350}>
      <LineChart data={data} margin={{ top: 5, right: 20, left: 10, bottom: 5 }}>
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis
          dataKey="date"
          tick={{ fontSize: 12 }}
          className="fill-muted-foreground"
          tickFormatter={(value) =>
            new Date(value).toLocaleDateString("de-DE", { day: "2-digit", month: "short" })
          }
        />
        <YAxis
          tick={{ fontSize: 12 }}
          className="fill-muted-foreground"
          tickFormatter={(value) => abbreviateNumber(value)}
        />
        <Tooltip content={<CustomTooltip />} />
        <Legend />
        <Line
          type="monotone"
          dataKey="revenue"
          stroke={CHART_COLORS.primary}
          strokeWidth={2}
          dot={false}
          activeDot={{ r: 5 }}
          name="Revenue"
        />
        <Line
          type="monotone"
          dataKey="target"
          stroke={CHART_COLORS.success}
          strokeWidth={2}
          strokeDasharray="5 5"
          dot={false}
          name="Target"
        />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

### Bar Chart

```tsx
import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  Legend,
} from "recharts";

export function ChannelBarChart({ data }: { data: { channel: string; spend: number; revenue: number }[] }) {
  return (
    <ResponsiveContainer width="100%" height={350}>
      <BarChart data={data} margin={{ top: 5, right: 20, left: 10, bottom: 5 }}>
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis dataKey="channel" tick={{ fontSize: 12 }} />
        <YAxis tickFormatter={(v) => abbreviateNumber(v)} />
        <Tooltip content={<CustomTooltip />} />
        <Legend />
        <Bar dataKey="spend" name="Spend" fill={CHART_COLORS.primary} radius={[4, 4, 0, 0]} />
        <Bar dataKey="revenue" name="Revenue" fill={CHART_COLORS.success} radius={[4, 4, 0, 0]} />
      </BarChart>
    </ResponsiveContainer>
  );
}
```

### Area Chart

```tsx
import {
  AreaChart,
  Area,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from "recharts";

export function TrafficAreaChart({ data }: { data: { date: string; organic: number; paid: number; direct: number }[] }) {
  return (
    <ResponsiveContainer width="100%" height={350}>
      <AreaChart data={data}>
        <defs>
          <linearGradient id="colorOrganic" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%" stopColor={CHART_COLORS.success} stopOpacity={0.3} />
            <stop offset="95%" stopColor={CHART_COLORS.success} stopOpacity={0} />
          </linearGradient>
          <linearGradient id="colorPaid" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%" stopColor={CHART_COLORS.primary} stopOpacity={0.3} />
            <stop offset="95%" stopColor={CHART_COLORS.primary} stopOpacity={0} />
          </linearGradient>
        </defs>
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis dataKey="date" />
        <YAxis />
        <Tooltip content={<CustomTooltip />} />
        <Area
          type="monotone"
          dataKey="organic"
          stackId="1"
          stroke={CHART_COLORS.success}
          fillOpacity={1}
          fill="url(#colorOrganic)"
        />
        <Area
          type="monotone"
          dataKey="paid"
          stackId="1"
          stroke={CHART_COLORS.primary}
          fillOpacity={1}
          fill="url(#colorPaid)"
        />
        <Area
          type="monotone"
          dataKey="direct"
          stackId="1"
          stroke={CHART_COLORS.orange}
          fill={CHART_COLORS.orange}
          fillOpacity={0.3}
        />
      </AreaChart>
    </ResponsiveContainer>
  );
}
```

### Pie / Donut Chart

```tsx
import { PieChart, Pie, Cell, Tooltip, ResponsiveContainer, Legend } from "recharts";

export function DistributionPieChart({
  data,
  innerRadius = 60,
}: {
  data: { name: string; value: number }[];
  innerRadius?: number; // 0 = pie, 60+ = donut
}) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <PieChart>
        <Pie
          data={data}
          cx="50%"
          cy="50%"
          innerRadius={innerRadius}
          outerRadius={100}
          paddingAngle={2}
          dataKey="value"
          label={({ name, percent }) => `${name} ${(percent * 100).toFixed(0)}%`}
        >
          {data.map((_, index) => (
            <Cell key={index} fill={SERIES_COLORS[index % SERIES_COLORS.length]} />
          ))}
        </Pie>
        <Tooltip content={<CustomTooltip />} />
        <Legend />
      </PieChart>
    </ResponsiveContainer>
  );
}
```

### Custom Tooltip

```tsx
// components/charts/custom-tooltip.tsx
interface TooltipProps {
  active?: boolean;
  payload?: Array<{ name: string; value: number; color: string }>;
  label?: string;
}

function CustomTooltip({ active, payload, label }: TooltipProps) {
  if (!active || !payload?.length) return null;

  return (
    <div className="rounded-lg border bg-background px-3 py-2 shadow-md">
      <p className="mb-1 text-sm font-medium">{label}</p>
      {payload.map((entry, index) => (
        <div key={index} className="flex items-center gap-2 text-sm">
          <div
            className="h-2.5 w-2.5 rounded-full"
            style={{ backgroundColor: entry.color }}
          />
          <span className="text-muted-foreground">{entry.name}:</span>
          <span className="font-medium">
            {typeof entry.value === "number"
              ? formatNumber(entry.value)
              : entry.value}
          </span>
        </div>
      ))}
    </div>
  );
}
```

---

## 3. Tremor -- Dashboard Components

### Installation

```bash
bun add @tremor/react
```

### KPI Cards

```tsx
import { Card, Metric, Text, Flex, BadgeDelta, ProgressBar } from "@tremor/react";

interface KpiCardProps {
  title: string;
  value: string;
  delta: string;
  deltaType: "increase" | "moderateIncrease" | "unchanged" | "moderateDecrease" | "decrease";
  progress?: number;
}

export function KpiCard({ title, value, delta, deltaType, progress }: KpiCardProps) {
  return (
    <Card>
      <Flex justifyContent="between" alignItems="center">
        <div>
          <Text>{title}</Text>
          <Metric>{value}</Metric>
        </div>
        <BadgeDelta deltaType={deltaType}>{delta}</BadgeDelta>
      </Flex>
      {progress !== undefined && (
        <ProgressBar value={progress} className="mt-3" />
      )}
    </Card>
  );
}

// Usage
<div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
  <KpiCard title="Revenue" value="$45,231" delta="+12.5%" deltaType="increase" progress={72} />
  <KpiCard title="Users" value="2,350" delta="+8.2%" deltaType="moderateIncrease" />
  <KpiCard title="Churn" value="3.2%" delta="-0.5%" deltaType="decrease" />
  <KpiCard title="ARPU" value="$19.25" delta="+2.1%" deltaType="increase" />
</div>
```

### Sparklines and Mini Charts

```tsx
import { SparkAreaChart, SparkBarChart, SparkLineChart } from "@tremor/react";

export function SparklineMetric({
  title,
  value,
  data,
  dataKey,
}: {
  title: string;
  value: string;
  data: any[];
  dataKey: string;
}) {
  return (
    <Card className="max-w-sm">
      <Text>{title}</Text>
      <Flex className="mt-2" justifyContent="between" alignItems="end">
        <Metric>{value}</Metric>
        <SparkAreaChart
          data={data}
          categories={[dataKey]}
          index="date"
          className="h-10 w-28"
          colors={["emerald"]}
        />
      </Flex>
    </Card>
  );
}
```

### Tremor Bar/Line Charts (Higher-Level API)

```tsx
import { BarChart as TremorBar, LineChart as TremorLine } from "@tremor/react";

// Tremor charts are higher-level -- less config, faster to build
<TremorBar
  data={data}
  index="month"
  categories={["Revenue", "Cost"]}
  colors={["blue", "rose"]}
  valueFormatter={(v) => `$${(v / 1000).toFixed(1)}K`}
  yAxisWidth={60}
/>

<TremorLine
  data={data}
  index="date"
  categories={["Sessions", "Conversions"]}
  colors={["emerald", "amber"]}
  curveType="monotone"
/>
```

---

## 4. Dashboard Layouts

### Grid Pattern for KPI Dashboard

```tsx
// app/(app)/dashboard/page.tsx
export default function DashboardPage() {
  return (
    <div className="space-y-6 p-6">
      {/* KPI row */}
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <KpiCard title="Revenue" value="$45,231" delta="+12.5%" deltaType="increase" />
        <KpiCard title="Orders" value="1,234" delta="+8.2%" deltaType="increase" />
        <KpiCard title="Conversion" value="3.2%" delta="+0.5%" deltaType="increase" />
        <KpiCard title="Avg. Order" value="$36.68" delta="-2.1%" deltaType="decrease" />
      </div>

      {/* Main charts row */}
      <div className="grid grid-cols-1 gap-6 lg:grid-cols-3">
        <div className="lg:col-span-2">
          <ChartCard title="Revenue Over Time" description="Last 30 days">
            <RevenueChart data={revenueData} />
          </ChartCard>
        </div>
        <div>
          <ChartCard title="Revenue by Channel">
            <DistributionPieChart data={channelData} />
          </ChartCard>
        </div>
      </div>

      {/* Secondary row */}
      <div className="grid grid-cols-1 gap-6 md:grid-cols-2">
        <ChartCard title="Ad Spend vs Revenue">
          <ChannelBarChart data={adData} />
        </ChartCard>
        <ChartCard title="Traffic Sources">
          <TrafficAreaChart data={trafficData} />
        </ChartCard>
      </div>

      {/* Table */}
      <ChartCard title="Top Campaigns">
        <CampaignTable data={campaigns} />
      </ChartCard>
    </div>
  );
}
```

### Reusable Chart Card Component

```tsx
// components/charts/chart-card.tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

interface ChartCardProps {
  title: string;
  description?: string;
  action?: React.ReactNode;
  children: React.ReactNode;
}

export function ChartCard({ title, description, action, children }: ChartCardProps) {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <div>
          <CardTitle className="text-base font-medium">{title}</CardTitle>
          {description && (
            <p className="text-sm text-muted-foreground">{description}</p>
          )}
        </div>
        {action}
      </CardHeader>
      <CardContent>{children}</CardContent>
    </Card>
  );
}
```

### Responsive Breakpoints Reference

| Breakpoint | Width | Tailwind | Layout |
|---|---|---|---|
| Mobile | < 640px | default | Single column, stacked cards |
| Tablet | 640-1023px | `sm:` | 2 columns for KPIs, single for charts |
| Desktop | 1024-1279px | `lg:` | 4 KPI cols, 2/3 + 1/3 for charts |
| Wide | 1280px+ | `xl:` | Full grid, side-by-side charts, sidebar |

---

## 5. Responsive Charts

### Mobile-Friendly Chart Pattern

```tsx
"use client";

import { useMediaQuery } from "@/hooks/use-media-query";

export function ResponsiveRevenueChart({ data }: { data: DataPoint[] }) {
  const isMobile = useMediaQuery("(max-width: 640px)");

  return (
    <ResponsiveContainer width="100%" height={isMobile ? 250 : 350}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis
          dataKey="date"
          tick={{ fontSize: isMobile ? 10 : 12 }}
          // Show fewer ticks on mobile
          interval={isMobile ? Math.floor(data.length / 5) : "preserveStartEnd"}
          angle={isMobile ? -45 : 0}
          textAnchor={isMobile ? "end" : "middle"}
          tickFormatter={(value) =>
            isMobile
              ? new Date(value).toLocaleDateString("de-DE", { day: "2-digit", month: "2-digit" })
              : new Date(value).toLocaleDateString("de-DE", { day: "2-digit", month: "short" })
          }
        />
        <YAxis
          tick={{ fontSize: isMobile ? 10 : 12 }}
          width={isMobile ? 40 : 60}
          tickFormatter={(v) => abbreviateNumber(v)}
        />
        <Tooltip content={<CustomTooltip />} />
        {/* Hide legend on mobile to save space */}
        {!isMobile && <Legend />}
        <Line
          type="monotone"
          dataKey="revenue"
          stroke={CHART_COLORS.primary}
          strokeWidth={2}
          dot={!isMobile} // Fewer dots on mobile
        />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

### useMediaQuery Hook

```tsx
// hooks/use-media-query.ts
"use client";

import { useState, useEffect } from "react";

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);

  useEffect(() => {
    const media = window.matchMedia(query);
    setMatches(media.matches);

    const listener = (e: MediaQueryListEvent) => setMatches(e.matches);
    media.addEventListener("change", listener);
    return () => media.removeEventListener("change", listener);
  }, [query]);

  return matches;
}
```

### Simplified Mobile View (Table Fallback)

```tsx
export function ChartWithTableFallback({ data }: { data: any[] }) {
  const isMobile = useMediaQuery("(max-width: 480px)");

  if (isMobile) {
    return (
      <div className="overflow-x-auto">
        <table className="w-full text-sm">
          <thead>
            <tr className="border-b text-left">
              <th className="pb-2">Date</th>
              <th className="pb-2 text-right">Value</th>
            </tr>
          </thead>
          <tbody>
            {data.slice(-7).map((row) => (
              <tr key={row.date} className="border-b">
                <td className="py-1.5">{row.date}</td>
                <td className="py-1.5 text-right font-medium">
                  {formatNumber(row.value)}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    );
  }

  return <TrafficAreaChart data={data} />;
}
```

---

## 6. Real-Time Charts

### Streaming Data with Supabase Realtime

```tsx
"use client";

import { useState, useEffect } from "react";
import { createClient } from "@/lib/supabase/client";
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  ResponsiveContainer,
  CartesianGrid,
} from "recharts";

interface LiveDataPoint {
  timestamp: string;
  value: number;
}

export function RealtimeChart({ metricId }: { metricId: string }) {
  const [data, setData] = useState<LiveDataPoint[]>([]);
  const maxPoints = 60; // Keep last 60 data points
  const supabase = createClient();

  useEffect(() => {
    // Load initial data
    supabase
      .from("metrics")
      .select("timestamp, value")
      .eq("metric_id", metricId)
      .order("timestamp", { ascending: false })
      .limit(maxPoints)
      .then(({ data }) => {
        if (data) setData(data.reverse());
      });

    // Subscribe to new data
    const channel = supabase
      .channel(`metrics:${metricId}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "metrics",
          filter: `metric_id=eq.${metricId}`,
        },
        (payload) => {
          setData((prev) => {
            const updated = [...prev, payload.new as LiveDataPoint];
            return updated.slice(-maxPoints);
          });
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [metricId]);

  return (
    <ResponsiveContainer width="100%" height={300}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis
          dataKey="timestamp"
          tickFormatter={(v) => new Date(v).toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" })}
        />
        <YAxis />
        <Line
          type="monotone"
          dataKey="value"
          stroke={CHART_COLORS.primary}
          strokeWidth={2}
          dot={false}
          isAnimationActive={false} // Disable for streaming performance
        />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

### Animated Value Change

```tsx
import { useSpring, animated } from "@react-spring/web";

export function AnimatedMetric({ value }: { value: number }) {
  const { number } = useSpring({
    from: { number: 0 },
    number: value,
    config: { duration: 800 },
  });

  return (
    <animated.span className="text-3xl font-bold">
      {number.to((n) => formatCurrency(n))}
    </animated.span>
  );
}
```

### Smooth Animation on Data Change

```tsx
// Key props for smooth updates:
<Line
  isAnimationActive={true}
  animationDuration={300}    // short for real-time feel
  animationEasing="ease-out"
/>

// For bar charts:
<Bar
  isAnimationActive={true}
  animationDuration={500}
  animationBegin={0}
/>
```

### Polling Fallback (When Realtime Is Not Available)

```tsx
import useSWR from "swr";

export function PollingChart({ endpoint }: { endpoint: string }) {
  const { data } = useSWR(endpoint, fetcher, {
    refreshInterval: 5000, // Poll every 5 seconds
  });

  return data ? <RevenueChart data={data} /> : <ChartSkeleton />;
}
```

---

## 7. Data Formatting

### Number Formatting Utilities

```ts
// lib/format.ts

/**
 * Format a number as currency.
 * @param value - The numeric value
 * @param abbreviated - Use abbreviations (1.2K, 3.4M)
 * @param currency - Currency code (default: EUR)
 */
export function formatCurrency(
  value: number,
  abbreviated = false,
  currency = "EUR"
): string {
  if (abbreviated) {
    return abbreviateNumber(value, currency);
  }

  return new Intl.NumberFormat("de-DE", {
    style: "currency",
    currency,
    minimumFractionDigits: 0,
    maximumFractionDigits: 2,
  }).format(value);
}

/**
 * Format a number as percentage.
 */
export function formatPercent(value: number, decimals = 1): string {
  return new Intl.NumberFormat("de-DE", {
    style: "percent",
    minimumFractionDigits: decimals,
    maximumFractionDigits: decimals,
  }).format(value / 100);
}

/**
 * Format a plain number with locale-aware separators.
 */
export function formatNumber(value: number, decimals = 0): string {
  return new Intl.NumberFormat("de-DE", {
    minimumFractionDigits: decimals,
    maximumFractionDigits: decimals,
  }).format(value);
}

/**
 * Abbreviate large numbers: 1200 -> 1.2K, 3400000 -> 3.4M
 */
export function abbreviateNumber(value: number, currency?: string): string {
  const abs = Math.abs(value);
  const sign = value < 0 ? "-" : "";
  const prefix = currency ? getCurrencySymbol(currency) : "";

  if (abs >= 1_000_000_000) {
    return `${sign}${prefix}${(abs / 1_000_000_000).toFixed(1)}B`;
  }
  if (abs >= 1_000_000) {
    return `${sign}${prefix}${(abs / 1_000_000).toFixed(1)}M`;
  }
  if (abs >= 1_000) {
    return `${sign}${prefix}${(abs / 1_000).toFixed(1)}K`;
  }
  return `${sign}${prefix}${abs.toFixed(0)}`;
}

function getCurrencySymbol(currency: string): string {
  return new Intl.NumberFormat("de-DE", { style: "currency", currency })
    .formatToParts(0)
    .find((p) => p.type === "currency")?.value ?? currency;
}

/**
 * Format a delta value with + or - prefix.
 */
export function formatDelta(value: number): { text: string; positive: boolean } {
  const prefix = value >= 0 ? "+" : "";
  return {
    text: `${prefix}${formatPercent(value)}`,
    positive: value >= 0,
  };
}

/**
 * Format duration (seconds to human-readable).
 */
export function formatDuration(seconds: number): string {
  if (seconds < 60) return `${seconds}s`;
  if (seconds < 3600) return `${Math.floor(seconds / 60)}m ${seconds % 60}s`;
  const h = Math.floor(seconds / 3600);
  const m = Math.floor((seconds % 3600) / 60);
  return `${h}h ${m}m`;
}
```

### Using Formatters in Charts

```tsx
<YAxis tickFormatter={(value) => abbreviateNumber(value)} />
<Tooltip formatter={(value: number) => [formatCurrency(value), "Revenue"]} />
```

---

## 8. Export -- PNG, CSV, PDF

### Chart to PNG (html2canvas)

```bash
bun add html2canvas-pro
```

```tsx
// components/charts/export-button.tsx
"use client";

import { useRef } from "react";
import html2canvas from "html2canvas-pro";
import { Button } from "@/components/ui/button";
import { Download } from "lucide-react";

export function ExportableChart({
  title,
  children,
}: {
  title: string;
  children: React.ReactNode;
}) {
  const chartRef = useRef<HTMLDivElement>(null);

  async function exportPNG() {
    if (!chartRef.current) return;

    const canvas = await html2canvas(chartRef.current, {
      backgroundColor: "#ffffff",
      scale: 2, // retina quality
    });

    const link = document.createElement("a");
    link.download = `${title.toLowerCase().replace(/\s+/g, "-")}-${new Date().toISOString().split("T")[0]}.png`;
    link.href = canvas.toDataURL("image/png");
    link.click();
  }

  return (
    <div>
      <div className="mb-2 flex justify-end">
        <Button variant="outline" size="sm" onClick={exportPNG}>
          <Download className="mr-1.5 h-3.5 w-3.5" />
          PNG
        </Button>
      </div>
      <div ref={chartRef}>{children}</div>
    </div>
  );
}
```

### Data to CSV

```ts
// lib/export-csv.ts

export function downloadCSV(
  data: Record<string, unknown>[],
  filename: string
) {
  if (!data.length) return;

  const headers = Object.keys(data[0]);
  const csvRows = [
    headers.join(","),
    ...data.map((row) =>
      headers
        .map((h) => {
          const val = row[h];
          const str = String(val ?? "");
          // Escape values containing commas or quotes
          return str.includes(",") || str.includes('"')
            ? `"${str.replace(/"/g, '""')}"`
            : str;
        })
        .join(",")
    ),
  ];

  const blob = new Blob([csvRows.join("\n")], { type: "text/csv;charset=utf-8;" });
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = `${filename}.csv`;
  link.click();
  URL.revokeObjectURL(link.href);
}
```

### PDF Export (jsPDF + autotable)

```bash
bun add jspdf jspdf-autotable html2canvas-pro
```

```tsx
import jsPDF from "jspdf";
import autoTable from "jspdf-autotable";
import html2canvas from "html2canvas-pro";

export async function exportDashboardPDF(
  chartRef: HTMLDivElement,
  tableData: Record<string, any>[],
  title: string
) {
  const pdf = new jsPDF("landscape", "mm", "a4");

  // Title
  pdf.setFontSize(18);
  pdf.text(title, 14, 22);
  pdf.setFontSize(10);
  pdf.text(`Generated: ${new Date().toLocaleDateString("de-DE")}`, 14, 30);

  // Chart as image
  const canvas = await html2canvas(chartRef, { scale: 2 });
  const imgData = canvas.toDataURL("image/png");
  pdf.addImage(imgData, "PNG", 14, 35, 267, 100);

  // Data table
  if (tableData.length) {
    const headers = Object.keys(tableData[0]);
    autoTable(pdf, {
      startY: 140,
      head: [headers],
      body: tableData.map((row) => headers.map((h) => String(row[h]))),
      styles: { fontSize: 8 },
    });
  }

  pdf.save(`${title.toLowerCase().replace(/\s+/g, "-")}.pdf`);
}
```

### PDF Report Generation (react-pdf -- Alternative)

```bash
bun add @react-pdf/renderer
```

```tsx
// components/reports/pdf-report.tsx
import {
  Document,
  Page,
  Text,
  View,
  StyleSheet,
  pdf,
} from "@react-pdf/renderer";

const styles = StyleSheet.create({
  page: { padding: 40, fontFamily: "Helvetica" },
  title: { fontSize: 24, fontWeight: "bold", marginBottom: 20 },
  kpiRow: { flexDirection: "row", gap: 16, marginBottom: 24 },
  kpiCard: { flex: 1, backgroundColor: "#f8f9fa", padding: 12, borderRadius: 4 },
  kpiLabel: { fontSize: 10, color: "#666" },
  kpiValue: { fontSize: 18, fontWeight: "bold", marginTop: 4 },
  row: { flexDirection: "row", borderBottomWidth: 1, borderBottomColor: "#eee", paddingVertical: 6 },
  cell: { flex: 1, fontSize: 10 },
  headerCell: { flex: 1, fontSize: 10, fontWeight: "bold" },
});

interface ReportData {
  title: string;
  dateRange: string;
  kpis: { label: string; value: string }[];
  tableHeaders: string[];
  tableRows: string[][];
}

function ReportDocument({ data }: { data: ReportData }) {
  return (
    <Document>
      <Page size="A4" style={styles.page}>
        <Text style={styles.title}>{data.title}</Text>
        <Text style={{ fontSize: 12, color: "#666", marginBottom: 24 }}>
          {data.dateRange}
        </Text>
        <View style={styles.kpiRow}>
          {data.kpis.map((kpi, i) => (
            <View key={i} style={styles.kpiCard}>
              <Text style={styles.kpiLabel}>{kpi.label}</Text>
              <Text style={styles.kpiValue}>{kpi.value}</Text>
            </View>
          ))}
        </View>
        <View>
          <View style={styles.row}>
            {data.tableHeaders.map((h, i) => (
              <Text key={i} style={styles.headerCell}>{h}</Text>
            ))}
          </View>
          {data.tableRows.map((row, i) => (
            <View key={i} style={styles.row}>
              {row.map((cell, j) => (
                <Text key={j} style={styles.cell}>{cell}</Text>
              ))}
            </View>
          ))}
        </View>
      </Page>
    </Document>
  );
}

export async function downloadPDF(data: ReportData, filename: string) {
  const blob = await pdf(<ReportDocument data={data} />).toBlob();
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.href = url;
  link.download = `${filename}.pdf`;
  link.click();
  URL.revokeObjectURL(url);
}
```

---

## 9. Accessible Charts

### ARIA Labels

```tsx
export function AccessibleChart({ data, title }: { data: DataPoint[]; title: string }) {
  return (
    <div role="img" aria-label={`${title} chart showing data from ${data[0]?.date} to ${data[data.length - 1]?.date}`}>
      <ResponsiveContainer width="100%" height={350}>
        <LineChart data={data} aria-hidden="true">
          {/* ... chart content ... */}
        </LineChart>
      </ResponsiveContainer>

      {/* Screen reader data table */}
      <table className="sr-only">
        <caption>{title}</caption>
        <thead>
          <tr>
            <th scope="col">Date</th>
            <th scope="col">Value</th>
          </tr>
        </thead>
        <tbody>
          {data.map((point, i) => (
            <tr key={i}>
              <td>{point.date}</td>
              <td>{point.value}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### Color-Blind Safe Palettes

```ts
// lib/chart-colors-accessible.ts

// Wong color-blind safe palette (works for all common types of color blindness)
export const COLOR_BLIND_SAFE = [
  "#0072B2", // blue
  "#E69F00", // orange
  "#009E73", // green
  "#CC79A7", // pink
  "#56B4E9", // light blue
  "#D55E00", // vermillion
  "#F0E442", // yellow
  "#000000", // black
];

// Use patterns in addition to color for critical distinctions.
// Combine with strokeDasharray for line charts:
const LINE_STYLES = [
  { strokeDasharray: "0" },          // solid
  { strokeDasharray: "8 4" },        // dashed
  { strokeDasharray: "2 2" },        // dotted
  { strokeDasharray: "8 4 2 4" },    // dash-dot
];

// Also use COLORBLIND_PALETTE from section 2 (OKLCH version)
```

### Data Table Alternative Toggle

```tsx
"use client";

import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Table, BarChart3 } from "lucide-react";

export function ChartWithTableToggle({
  data,
  chart,
  columns,
}: {
  data: Record<string, unknown>[];
  chart: React.ReactNode;
  columns: { key: string; label: string }[];
}) {
  const [view, setView] = useState<"chart" | "table">("chart");

  return (
    <div>
      <div className="mb-2 flex justify-end gap-1">
        <Button
          variant={view === "chart" ? "secondary" : "ghost"}
          size="icon"
          onClick={() => setView("chart")}
          aria-label="Show chart"
        >
          <BarChart3 className="h-4 w-4" />
        </Button>
        <Button
          variant={view === "table" ? "secondary" : "ghost"}
          size="icon"
          onClick={() => setView("table")}
          aria-label="Show data table"
        >
          <Table className="h-4 w-4" />
        </Button>
      </div>

      {view === "chart" ? (
        chart
      ) : (
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b">
                {columns.map((col) => (
                  <th key={col.key} className="px-3 py-2 text-left font-medium">
                    {col.label}
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {data.map((row, i) => (
                <tr key={i} className="border-b">
                  {columns.map((col) => (
                    <td key={col.key} className="px-3 py-2">
                      {String(row[col.key] ?? "")}
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
}
```

---

## 10. Common Chart Types -- When to Use

| Chart Type | Use Case | Recharts Component | Data Shape |
|---|---|---|---|
| **Line** | Trends over time | `LineChart` + `Line` | `[{ date, value }]` |
| **Bar** | Category comparison | `BarChart` + `Bar` | `[{ name, valueA, valueB }]` |
| **Area** | Volume/composition over time | `AreaChart` + `Area` | `[{ date, organic, paid }]` |
| **Pie / Donut** | Part-to-whole composition | `PieChart` + `Pie` | `[{ name, value }]` |
| **Stacked Bar** | Composition across categories | `BarChart` + `Bar` (stackId) | `[{ name, a, b, c }]` |
| **Funnel** | Conversion steps | `FunnelChart` + `Funnel` | `[{ name, value }]` |
| **Radar** | Multi-variable comparison | `RadarChart` + `Radar` | `[{ subject, a, b }]` |
| **Scatter** | Correlation between variables | `ScatterChart` + `Scatter` | `[{ x, y }]` |
| **Composed** | Mixed chart types | `ComposedChart` | Bar + Line overlay |
| **Heatmap** | Density / intensity matrix | Custom (CSS grid) | `number[][]` |

### Funnel Chart

```tsx
import { FunnelChart, Funnel, Cell, Tooltip, LabelList, ResponsiveContainer } from "recharts";

const funnelData = [
  { name: "Visitors", value: 10000 },
  { name: "Sign-ups", value: 3500 },
  { name: "Activated", value: 1800 },
  { name: "Subscribed", value: 650 },
  { name: "Retained (M3)", value: 480 },
];

export function ConversionFunnel({ data }: { data: typeof funnelData }) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <FunnelChart>
        <Tooltip content={<CustomTooltip />} />
        <Funnel dataKey="value" data={data} isAnimationActive>
          {data.map((_, index) => (
            <Cell key={index} fill={SERIES_COLORS[index % SERIES_COLORS.length]} />
          ))}
          <LabelList
            position="right"
            fill="#666"
            stroke="none"
            dataKey="name"
            fontSize={12}
          />
        </Funnel>
      </FunnelChart>
    </ResponsiveContainer>
  );
}
```

### Composed Chart (Bar + Line Overlay)

```tsx
import {
  ComposedChart,
  Bar,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
} from "recharts";

export function SpendRevenueChart({
  data,
}: {
  data: { month: string; spend: number; revenue: number; roas: number }[];
}) {
  return (
    <ResponsiveContainer width="100%" height={350}>
      <ComposedChart data={data}>
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis dataKey="month" />
        <YAxis yAxisId="left" tickFormatter={(v) => abbreviateNumber(v)} />
        <YAxis yAxisId="right" orientation="right" tickFormatter={(v) => `${v}x`} />
        <Tooltip content={<CustomTooltip />} />
        <Legend />
        <Bar yAxisId="left" dataKey="spend" fill={`${CHART_COLORS.primary} / 0.7)`} name="Spend" radius={[4, 4, 0, 0]} />
        <Bar yAxisId="left" dataKey="revenue" fill={`${CHART_COLORS.success} / 0.7)`} name="Revenue" radius={[4, 4, 0, 0]} />
        <Line
          yAxisId="right"
          type="monotone"
          dataKey="roas"
          stroke={CHART_COLORS.orange}
          strokeWidth={2}
          dot={{ r: 3 }}
          name="ROAS"
        />
      </ComposedChart>
    </ResponsiveContainer>
  );
}
```

### Heatmap (Custom with CSS Grid)

Recharts does not have a native heatmap. Build one with a CSS grid:

```tsx
// components/charts/heatmap.tsx
"use client";

import { Fragment } from "react";

interface HeatmapProps {
  data: { x: string; y: string; value: number }[];
  xLabels: string[];
  yLabels: string[];
  maxValue: number;
}

export function Heatmap({ data, xLabels, yLabels, maxValue }: HeatmapProps) {
  function getColor(value: number): string {
    const intensity = Math.min(value / maxValue, 1);
    // OKLCH: low value = light, high value = saturated blue
    return `oklch(${0.95 - intensity * 0.35} ${intensity * 0.2} 250)`;
  }

  const dataMap = new Map(data.map((d) => [`${d.x}-${d.y}`, d.value]));

  return (
    <div className="overflow-x-auto">
      <div className="inline-grid gap-1" style={{ gridTemplateColumns: `80px repeat(${xLabels.length}, 48px)` }}>
        {/* Header row */}
        <div />
        {xLabels.map((x) => (
          <div key={x} className="truncate text-center text-xs text-muted-foreground">
            {x}
          </div>
        ))}

        {/* Data rows */}
        {yLabels.map((y) => (
          <Fragment key={y}>
            <div className="flex items-center text-xs text-muted-foreground">
              {y}
            </div>
            {xLabels.map((x) => {
              const value = dataMap.get(`${x}-${y}`) ?? 0;
              return (
                <div
                  key={`${x}-${y}`}
                  className="flex h-10 w-12 items-center justify-center rounded text-xs font-medium"
                  style={{ backgroundColor: getColor(value) }}
                  title={`${x}, ${y}: ${value}`}
                >
                  {value > 0 ? value : ""}
                </div>
              );
            })}
          </Fragment>
        ))}
      </div>
    </div>
  );
}
```

---

## Quick Decision Guide

- **Need a chart fast?** Use Tremor (pre-styled, minimal config).
- **Need full control?** Use Recharts (composable, customizable).
- **Building a KPI dashboard?** Tremor KPI cards + Recharts for detailed charts.
- **Real-time data?** Recharts + Supabase Realtime, disable animation for streaming.
- **Exporting reports?** html2canvas-pro for charts, jsPDF + autotable for PDF, @react-pdf/renderer for styled PDFs.
- **10K+ data points?** Consider Chart.js (canvas) or virtualized approaches.
- **Accessibility?** Always include sr-only data table, use COLORBLIND_PALETTE, vary line dash patterns.
