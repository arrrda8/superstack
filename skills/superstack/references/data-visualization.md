# Data Visualization

Complete guide for building charts, dashboards, and data visualizations in Next.js. Primary recommendation: **Recharts**.

---

## 1. Library Comparison

| Criteria | Recharts | Tremor | Chart.js (react-chartjs-2) | Nivo |
|---|---|---|---|---|
| Bundle size (gzip) | ~45 KB | ~90 KB (includes Tremor UI) | ~35 KB | ~60 KB per chart type |
| Customization | High (composable API) | Low-Medium (opinionated) | Medium | Very High |
| SSR support | Good (with dynamic import) | Good (built for Next.js) | Poor (canvas-based) | Good (SVG mode) |
| TypeScript | Excellent | Excellent | Good | Excellent |
| Animation | Built-in | Built-in | Built-in | Built-in |
| Learning curve | Low | Very Low | Medium | Medium-High |
| React-native feel | Declarative JSX | Pre-built components | Imperative config | Declarative JSX |
| Accessibility | Manual | Basic built-in | Poor | Good |
| Active maintenance | Active | Active | Active | Active |

**Decision guide:**
- **Recharts** -- Best all-around for custom dashboards. Composable, flexible, good docs.
- **Tremor** -- Best for shipping fast. Pre-built dashboard components. Less control.
- **Chart.js** -- Best if you need canvas rendering (large datasets, 10K+ points). Poor SSR.
- **Nivo** -- Best for advanced/exotic chart types (sunburst, chord, sankey). Heavier.

**Recommendation:** Use Recharts for most projects. Add Tremor for rapid prototyping or internal tools.

---

## 2. Recharts Setup

### Installation

```bash
bun add recharts
```

### Dynamic Import (SSR-safe)

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

interface RevenueChartProps {
  data: DataPoint[];
}

export function RevenueChart({ data }: RevenueChartProps) {
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
          tickFormatter={(value) => formatCurrency(value, true)}
        />
        <Tooltip content={<CustomTooltip />} />
        <Legend />
        <Line
          type="monotone"
          dataKey="revenue"
          stroke="oklch(0.65 0.2 250)"
          strokeWidth={2}
          dot={false}
          activeDot={{ r: 5 }}
          name="Revenue"
        />
        <Line
          type="monotone"
          dataKey="target"
          stroke="oklch(0.7 0.15 150)"
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
  Cell,
} from "recharts";

const COLORS = [
  "oklch(0.65 0.2 250)",  // blue
  "oklch(0.7 0.18 150)",  // green
  "oklch(0.7 0.2 30)",    // orange
  "oklch(0.65 0.22 330)", // pink
  "oklch(0.75 0.15 80)",  // yellow
];

export function ChannelBarChart({ data }: { data: { channel: string; spend: number; revenue: number }[] }) {
  return (
    <ResponsiveContainer width="100%" height={350}>
      <BarChart data={data} margin={{ top: 5, right: 20, left: 10, bottom: 5 }}>
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis dataKey="channel" tick={{ fontSize: 12 }} />
        <YAxis tickFormatter={(v) => formatCurrency(v, true)} />
        <Tooltip content={<CustomTooltip />} />
        <Bar dataKey="spend" name="Spend" fill="oklch(0.65 0.2 250)" radius={[4, 4, 0, 0]} />
        <Bar dataKey="revenue" name="Revenue" fill="oklch(0.7 0.18 150)" radius={[4, 4, 0, 0]} />
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
        <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
        <XAxis dataKey="date" />
        <YAxis />
        <Tooltip content={<CustomTooltip />} />
        <Area
          type="monotone"
          dataKey="organic"
          stackId="1"
          stroke="oklch(0.7 0.18 150)"
          fill="oklch(0.7 0.18 150 / 0.3)"
        />
        <Area
          type="monotone"
          dataKey="paid"
          stackId="1"
          stroke="oklch(0.65 0.2 250)"
          fill="oklch(0.65 0.2 250 / 0.3)"
        />
        <Area
          type="monotone"
          dataKey="direct"
          stackId="1"
          stroke="oklch(0.7 0.2 30)"
          fill="oklch(0.7 0.2 30 / 0.3)"
        />
      </AreaChart>
    </ResponsiveContainer>
  );
}
```

### Pie Chart

```tsx
import { PieChart, Pie, Cell, Tooltip, ResponsiveContainer, Legend } from "recharts";

export function DistributionPieChart({ data }: { data: { name: string; value: number }[] }) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <PieChart>
        <Pie
          data={data}
          cx="50%"
          cy="50%"
          innerRadius={60}
          outerRadius={100}
          paddingAngle={2}
          dataKey="value"
          label={({ name, percent }) => `${name} ${(percent * 100).toFixed(0)}%`}
        >
          {data.map((_, index) => (
            <Cell key={index} fill={COLORS[index % COLORS.length]} />
          ))}
        </Pie>
        <Tooltip />
        <Legend />
      </PieChart>
    </ResponsiveContainer>
  );
}
```

### Custom Tooltip

```tsx
// components/charts/custom-tooltip.tsx
function CustomTooltip({ active, payload, label }: any) {
  if (!active || !payload?.length) return null;

  return (
    <div className="rounded-lg border bg-background px-3 py-2 shadow-md">
      <p className="mb-1 text-sm font-medium">{label}</p>
      {payload.map((entry: any, index: number) => (
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

### OKLCH Color Palette for Charts

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

// Ordered palette for series
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
```

---

## 3. Tremor

### Installation

```bash
bun add @tremor/react
```

### KPI Cards

```tsx
import { Card, Metric, Text, Flex, BadgeDelta } from "@tremor/react";

interface KpiCardProps {
  title: string;
  value: string;
  delta: string;
  deltaType: "increase" | "moderateIncrease" | "unchanged" | "moderateDecrease" | "decrease";
}

export function KpiCard({ title, value, delta, deltaType }: KpiCardProps) {
  return (
    <Card>
      <Flex justifyContent="between" alignItems="center">
        <div>
          <Text>{title}</Text>
          <Metric>{value}</Metric>
        </div>
        <BadgeDelta deltaType={deltaType}>{delta}</BadgeDelta>
      </Flex>
    </Card>
  );
}

// Usage
<div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
  <KpiCard title="Revenue" value="$45,231" delta="+12.5%" deltaType="increase" />
  <KpiCard title="Users" value="2,350" delta="+8.2%" deltaType="moderateIncrease" />
  <KpiCard title="Churn" value="3.2%" delta="-0.5%" deltaType="decrease" />
  <KpiCard title="ARPU" value="$19.25" delta="+2.1%" deltaType="increase" />
</div>
```

### Tremor Sparklines and Mini Charts

```tsx
import { SparkAreaChart, SparkBarChart, SparkLineChart } from "@tremor/react";

export function MiniTrend({ data }: { data: { value: number }[] }) {
  return (
    <SparkAreaChart
      data={data}
      categories={["value"]}
      index="date"
      colors={["emerald"]}
      className="h-8 w-24"
    />
  );
}
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
        <KpiCard title="Revenue" value="$45,231" delta="+12.5%" />
        <KpiCard title="Orders" value="1,234" delta="+8.2%" />
        <KpiCard title="Conversion" value="3.2%" delta="+0.5%" />
        <KpiCard title="Avg. Order" value="$36.68" delta="-2.1%" />
      </div>

      {/* Main charts row */}
      <div className="grid grid-cols-1 gap-4 lg:grid-cols-3">
        <div className="lg:col-span-2">
          <ChartCard title="Revenue Over Time">
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
      <div className="grid grid-cols-1 gap-4 lg:grid-cols-2">
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

### Chart Card Component

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

| Breakpoint | Width | Layout |
|---|---|---|
| Mobile | < 640px | Single column, stacked cards |
| Tablet | 640-1023px | 2 columns for KPIs, single for charts |
| Desktop | 1024-1279px | 4 KPI cols, 2/3 + 1/3 for charts |
| Wide | 1280px+ | Full grid, side-by-side charts |

---

## 5. Responsive Charts

### Mobile-Friendly Chart Patterns

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
          tickFormatter={(value) =>
            isMobile
              ? new Date(value).toLocaleDateString("de-DE", { day: "2-digit", month: "2-digit" })
              : new Date(value).toLocaleDateString("de-DE", { day: "2-digit", month: "short" })
          }
        />
        <YAxis
          tick={{ fontSize: isMobile ? 10 : 12 }}
          width={isMobile ? 40 : 60}
          tickFormatter={(v) => formatCurrency(v, true)}
        />
        <Tooltip content={<CustomTooltip />} />
        {/* Hide legend on mobile to save space */}
        {!isMobile && <Legend />}
        <Line
          type="monotone"
          dataKey="revenue"
          stroke="oklch(0.65 0.2 250)"
          strokeWidth={2}
          dot={false}
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

---

## 6. Real-Time Charts

### Streaming Data with Supabase Realtime

```tsx
"use client";

import { useState, useEffect, useRef } from "react";
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
            // Keep only last N points
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
          stroke="oklch(0.65 0.2 250)"
          strokeWidth={2}
          dot={false}
          isAnimationActive={true}
          animationDuration={300}
          animationEasing="ease-out"
        />
      </LineChart>
    </ResponsiveContainer>
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

// For bar charts, use:
<Bar
  isAnimationActive={true}
  animationDuration={500}
  animationBegin={0}
/>
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
 * Format a delta value with + or - prefix and color hint.
 */
export function formatDelta(value: number): { text: string; positive: boolean } {
  const prefix = value >= 0 ? "+" : "";
  return {
    text: `${prefix}${formatPercent(value)}`,
    positive: value >= 0,
  };
}
```

---

## 8. Export

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
      <div className="flex justify-end mb-2">
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
          // Escape values containing commas or quotes
          const str = String(val ?? "");
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

### PDF Report Generation (react-pdf)

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
  section: { marginBottom: 16 },
  sectionTitle: { fontSize: 14, fontWeight: "bold", marginBottom: 8 },
  row: { flexDirection: "row", borderBottomWidth: 1, borderBottomColor: "#eee", paddingVertical: 6 },
  cell: { flex: 1, fontSize: 10 },
  headerCell: { flex: 1, fontSize: 10, fontWeight: "bold" },
  kpiRow: { flexDirection: "row", gap: 16, marginBottom: 24 },
  kpiCard: { flex: 1, backgroundColor: "#f8f9fa", padding: 12, borderRadius: 4 },
  kpiLabel: { fontSize: 10, color: "#666" },
  kpiValue: { fontSize: 18, fontWeight: "bold", marginTop: 4 },
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

        {/* KPI Cards */}
        <View style={styles.kpiRow}>
          {data.kpis.map((kpi, i) => (
            <View key={i} style={styles.kpiCard}>
              <Text style={styles.kpiLabel}>{kpi.label}</Text>
              <Text style={styles.kpiValue}>{kpi.value}</Text>
            </View>
          ))}
        </View>

        {/* Data Table */}
        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Details</Text>
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

export async function generatePDF(data: ReportData): Promise<Blob> {
  return await pdf(<ReportDocument data={data} />).toBlob();
}

// Trigger download
export async function downloadPDF(data: ReportData, filename: string) {
  const blob = await generatePDF(data);
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
            <th>Date</th>
            <th>Value</th>
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

// Use patterns in addition to color for critical distinctions
// Combine with strokeDasharray for line charts:
const LINE_STYLES = [
  { strokeDasharray: "0" },          // solid
  { strokeDasharray: "8 4" },        // dashed
  { strokeDasharray: "2 2" },        // dotted
  { strokeDasharray: "8 4 2 4" },    // dash-dot
];
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
      <div className="flex justify-end gap-1 mb-2">
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

| Chart Type | Use Case | Recharts Component | Example |
|---|---|---|---|
| **Line** | Trends over time | `LineChart` + `Line` | Revenue over months, daily active users |
| **Bar** | Category comparison | `BarChart` + `Bar` | Revenue by channel, spend by campaign |
| **Area** | Volume/composition over time | `AreaChart` + `Area` | Traffic sources stacked, cumulative revenue |
| **Pie / Donut** | Part-to-whole composition | `PieChart` + `Pie` | Budget allocation, traffic share |
| **Funnel** | Conversion steps | `FunnelChart` + `Funnel` | Signup funnel, sales pipeline |
| **Radar** | Multi-variable comparison | `RadarChart` + `Radar` | Feature comparison, skill assessment |
| **Scatter** | Correlation between variables | `ScatterChart` + `Scatter` | Spend vs. revenue correlation |
| **Composed** | Mixed chart types | `ComposedChart` | Bar + Line overlay (volume + trend) |

### Funnel Chart Example

```tsx
import { FunnelChart, Funnel, Cell, Tooltip, LabelList, ResponsiveContainer } from "recharts";

const funnelData = [
  { name: "Visitors", value: 10000 },
  { name: "Sign-ups", value: 3500 },
  { name: "Activated", value: 1800 },
  { name: "Subscribed", value: 650 },
  { name: "Retained (M3)", value: 480 },
];

const FUNNEL_COLORS = [
  "oklch(0.65 0.2 250)",
  "oklch(0.68 0.19 230)",
  "oklch(0.71 0.18 200)",
  "oklch(0.74 0.17 170)",
  "oklch(0.7 0.18 150)",
];

export function ConversionFunnel() {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <FunnelChart>
        <Tooltip />
        <Funnel dataKey="value" data={funnelData} isAnimationActive>
          {funnelData.map((_, index) => (
            <Cell key={index} fill={FUNNEL_COLORS[index]} />
          ))}
          <LabelList
            position="right"
            content={({ name, value }) => (
              <text x={0} y={0} fill="#666" fontSize={12}>
                {name}: {formatNumber(value as number)}
              </text>
            )}
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
        <YAxis yAxisId="left" tickFormatter={(v) => formatCurrency(v, true)} />
        <YAxis yAxisId="right" orientation="right" tickFormatter={(v) => `${v}x`} />
        <Tooltip content={<CustomTooltip />} />
        <Legend />
        <Bar yAxisId="left" dataKey="spend" fill="oklch(0.65 0.2 250 / 0.7)" name="Spend" radius={[4, 4, 0, 0]} />
        <Bar yAxisId="left" dataKey="revenue" fill="oklch(0.7 0.18 150 / 0.7)" name="Revenue" radius={[4, 4, 0, 0]} />
        <Line
          yAxisId="right"
          type="monotone"
          dataKey="roas"
          stroke="oklch(0.7 0.2 30)"
          strokeWidth={2}
          dot={{ r: 3 }}
          name="ROAS"
        />
      </ComposedChart>
    </ResponsiveContainer>
  );
}
```

### Heatmap (Custom with Recharts)

Recharts does not have a native heatmap. Build one with a grid of `<rect>` elements or use Nivo's `HeatMap`. Simple approach with Tailwind:

```tsx
// components/charts/heatmap.tsx
"use client";

interface HeatmapProps {
  data: { x: string; y: string; value: number }[];
  xLabels: string[];
  yLabels: string[];
  maxValue: number;
}

export function Heatmap({ data, xLabels, yLabels, maxValue }: HeatmapProps) {
  function getColor(value: number): string {
    const intensity = Math.min(value / maxValue, 1);
    // oklch: low value = light, high value = saturated blue
    return `oklch(${0.95 - intensity * 0.35} ${intensity * 0.2} 250)`;
  }

  const dataMap = new Map(data.map((d) => [`${d.x}-${d.y}`, d.value]));

  return (
    <div className="overflow-x-auto">
      <div className="inline-grid gap-1" style={{ gridTemplateColumns: `80px repeat(${xLabels.length}, 48px)` }}>
        {/* Header row */}
        <div />
        {xLabels.map((x) => (
          <div key={x} className="text-center text-xs text-muted-foreground truncate">
            {x}
          </div>
        ))}

        {/* Data rows */}
        {yLabels.map((y) => (
          <>
            <div key={`label-${y}`} className="flex items-center text-xs text-muted-foreground">
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
          </>
        ))}
      </div>
    </div>
  );
}
```
